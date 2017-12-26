### 什么是Block
Block是C语言的扩充功能，很多资料都用这样一句话来定义Block：`带有自动变量（局部变量）的匿名函数`。

其中匿名函数很好理解，就是不带有名称的函数。“带有自动变量”则是描述了一种能力，就是可以捕获自动变量的值，保存该自动变量的瞬间值，即便改写了Block中使用的自动变量的值也不会影响Block执行时自动变量的值。当然，如果想让Block具备改写自动变量的值的能力，可以在声明该自动变量时可以用__block修饰符修饰，具体实现方式稍后会讲到。

关于Block的语法，请使劲戳这里→[fuckingblocksyntax.com](http://fuckingblocksyntax.com/)

### Block的实现原理

我们知道，OC是C语言的超集，OC中的对象实际上是由struct+isa指针实现的，通过clang编译器可以发现Block的数据结构，实际上也是由struct+isa指针来实现的，所以block实际上就是OC对象。

```
//这个是源代码
int main(){
    void (^blk)(void) = ^{printf("Block\n");};
    block();
    return 0;
}
```
编译之后的代码如下：

```
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};

struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0 *Desc;

    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc,int flags=0){
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
}
};

struct void __main_block_func_0(struct __main_block_impl_0 *__cself){
    printf("Block\n");
}

static struct __main_block_desc_0{
    unsigned long reserved;
    unsigned long Block_size;
} __main_block_desc_0_DATA = {
    0,
    sizeof(struct __main_block_impl_0)
};
```
显然一个block对象被编译成了一个\_\_main\_block\_impl\_0类型的结构体，而这个结构体又是由另外两个结构体加上一个构造函数构成的。其中\_\_block\_impl有一个函数指针，指向block内具体的代码\_\_main\_block\_func\_0。如下图所示

![Block数据结构](http://img.blog.csdn.net/20150727171855784)

### Block的存储域
根据上面Block的构造函数\_\_main\_block\_impl\_0
> impl.isa = &_NSConcreteStackBlock

block中的isa指向的是该block的Class。在block runtime中，定义了6种类：

_NSConcreteStackBlock     栈上创建的block

_NSConcreteMallocBlock  堆上创建的block

_NSConcreteGlobalBlock   作为全局变量的block

_NSConcreteWeakBlockVariable

_NSConcreteAutoBlock

_NSConcreteFinalizingBlock

我们能用到的是前三种。从命名就可以看出这三个Block类的差别主要在存储的区域不同。如下表所示：

| 类    | 设置对象的存储域                                                |
| ------------ | ---------------------------------------------------|
| _NSConcreteStackBlock   | 栈           |
| _NSConcreteGlobalBlock    | 程序的数据区域（.data区）     |
| _NSConcreteMallocBlock    | 堆                 |

当block体没有使用自动变量时，不存在对自动变量的截获，因此block结构体实例的内容不依赖执行时的状态，所以整个程序中只需要一个实例。因此将block用结构体实例设置在与全局变量相同的数据区域中即可。如下面这个block就是一个典型的NSConcreteGlobalBlock：
> void (^blk)(void) = ^{printf("Global Block!");};

而当block有捕获自动变量时，struct第一次被创建时，它是存在于该函数的栈帧上的，其Class是固定的_NSConcreteStackBlock。其捕获的变量是会赋值到结构体的成员上，所以当block初始化完成后，捕获到的变量不能更改。

当函数返回时，函数的栈帧被销毁，这个block的内存也会被清除。所以函数结束后仍需要此block时，就必须用Block_copy()方法将它拷贝到堆上。这个方法所做的处理就是：申请内存，将栈上的数据复制过去，将isa指针指向_NSConcreteMallocBlock，最后向捕获的对象发送retain。增加block引用计数。

在非arc的情况下，copy的操作需要程序员自行在合适的地方加上copy的操作，而在arc之后，编译器会智能的在适当的地方加上copy的操作，多数情况下在arc的环境下程序员不需要对block进行内存管理。但是也存在例外，比如下面这个例子：

```
- (id)getBlockArray
{
	int val = 10；
	return [[NSArray alloc] initWithObjects:
			^{NSLog(@"blk0:%d",val);}],
			^{NSLog(@"blk1:%d",val);}],nil];
}

id obj = getBlockArray();
typedef void (^blk_t)(void);
blk_t blk = (blk_t)[obj objectAtIndex:0];
blk();
```
运行后会发现blk()处会发生崩溃，这是由于在getBlockArray执行结束时，栈上的block已经被废弃的缘故。此处即便是arc编译器也不能判断是否需要复制。手动对调用block对象调用copy就会正常运行。所以在arc下也不能保证用到的所有block都是在堆上，生命周期都能在程序员的控制之下。

那如果对于已经配置在堆上或者数据区域的block调用copy方法又会如何呢？如下表所示：

| Block的类    | 副本源的配置存储域  |  复制效果                                              |
| ------------ | ----------------------------|------------ |
| _NSConcreteStackBlock   | 栈           | 从栈复制到堆 |
| _NSConcreteGlobalBlock    | 程序的数据区域（.data区） | 什么也不做 |
| _NSConcreteMallocBlock    | 堆                 | 引用计数增加 |

由此看出，无论Block配置在何处，用copy方法都不会引起任何问题。在不确定时调用copy方法即可。

### Block捕获自动变量
> 对于非OC对象block的捕获

```
typedef void (^Block)(void);

Block block;
{
    int val = 0;
    block = ^(){
        NSLog(@"val = %d",val);
    };
}
block();
```
用clang编译之后（为简化代码，只贴出改变的部分）

```
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0 *Desc;
    int val;

    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc,int _val, int flags=0) : val(_val){
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
}
};

struct void __main_block_func_0(struct __main_block_impl_0 *__cself){
    int val = __cself->val;
    printf("val = %d",val);
}
```
可以看出，当block要捕获自动变量的时候，会在\_\_main\_block\_impl\_0结构体增加一个成员变量，并在结构体的构造函数中对变量赋值。

当block被执行时，会把block对象也就是\_\_main\_block\_impl\_0结构体传入\_\_main\_block\_func\_0结构体中，取出自动变量的值，进行操作。

这样的结构决定了block捕获的值始终是自动变量被捕获时瞬间的值，即便自动变量的值在后面改变，再调用block，block的结构已经确定，捕获的自动变量的值也就不会改变了。

如果要让block捕获的自动变量随着自动变量本身的改变而改变的话，需要用到__block修饰符。

> __block修饰符是如何做到修改变量值的呢？

如果把val变量加上__block修饰符，编译器会怎么做呢？

```
//int val = 0; 原代码
__block int val = 0;//修改后的代码

```
编译后的代码：

```
struct __Block_byref_val_0 {
    void *__isa;
    __Block_byref_val_0 *forwarding;
    int __flags;
    int __size;
    int val;
};

struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0 *Desc;
    __Block_byref_val_0 *val;

    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc,__Block_byref_val_0 *_val, int flags=0) : val(_val->__forwrding){
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
}
};

struct void __main_block_func_0(struct __main_block_impl_0 *__cself){
    __Block_byref_val_0 *val = __cself->val;
    printf("val = %d",val->__forwarding->val);
}
```
可以看到，对于用__block修饰过的自动变量，block不会再简单地把捕获的自动变量当做成员变量了，而是增加了一个首地址是isa结构体指针。

当调用block的时候，再去取捕获的自动变量的值的时候获取到的是结构体的地址，只要把地址传递过去，就有了最高的操作权限，到时候再去取值就可以取到内存中最新的值。这也就是为什么__block修饰符能够让block拥有改变自动变量值的原因。

> 那么对于OC对象的捕获又是如何呢？


```
typedef void (^blk_t)(id obj);
blk_t blk;
{
	id mArray = [[NSMutableArray alloc] init];
    blk = ^(id obj) {
        [mArray addObject:obj];
        NSLog(@"array count = %ld",[mArray count]);
    };
}
blk([[NSObject alloc] init]);
```
clang编译后如下：

```
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0 *Desc;
    id __strong mArray;

    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc,id __strong mArray, int flags=0) : val(_val->__forwrding){
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
}
};

struct void __main_block_func_0(struct __main_block_impl_0 *__cself){
    id __strong mArray = __cself->val;
    [mArray addObject:obj]
    NSLog(@"array count = %ld",[mArray count]);
}

static void __ViewController__setBlock_block_copy_0(struct __ViewController__setBlock_block_impl_0*dst, struct __ViewController__setBlock_block_impl_0*src) {
_Block_object_assign((void*)&dst->mArray, (void*)src->mArray, 3/*BLOCK_FIELD_IS_OBJECT*/);
}

static void __ViewController__setBlock_block_dispose_0(struct __ViewController__setBlock_block_impl_0*src) {
_Block_object_dispose((void*)src->mArray, 3/*BLOCK_FIELD_IS_OBJECT*/);
}

static struct __ViewController__setBlock_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __ViewController__setBlock_block_impl_0*, struct __ViewController__setBlock_block_impl_0*);
  void (*dispose)(struct __ViewController__setBlock_block_impl_0*);
}

```

可以看到，被block截获的自动变量mArray，变成了block结构体中带有__strong修饰符的成员变量。而desc结构体中多出了copy和dispose两个方法，这是由于c语言结构体对于oc引用类型不能很好地内存管理。但是oc的运行时库能准确掌握block从栈复制到堆以及堆上block被废弃的时机。因此copy和dispose两个方法就是对引用类型进行内存管理的方法。

如果将源代码中mArray在block体里面重新赋值又会怎样呢？

```
typedef void (^blk_t)(id obj);
blk_t blk;
{
	id mArray = [[NSMutableArray alloc] init];
    blk = ^(id obj) {
        mArray =  [[NSMutableArray alloc] init];
        NSLog(@"array count = %ld",[mArray count]);
    };
}
blk([[NSObject alloc] init]);
```
这样的话编译器会报错。由此可见无论是基础数据类型还是引用类型，都遵循着可以访问，不可修改（赋值）的原则。

如果再加上__block，结构又会如何呢?

由于篇幅限制，就不再贴代码了。结果是，和基础类型加__block修饰符一样，block结构体会添加一个首地址是isa指针的结构体指针，这个结构体里面除了保存有自动变量的数据之外，还有引用类型才有的内存管理方法copy和dispose。

### Block循环引用
如果再Block中使用附有__strong修饰符的对象类型自动变量，那么当Block从栈复制到堆上的时候，该对象被Block持有。这样就会引起循环引用。为了解决循环一个引用通常会用weak来生命self。

另外，ARC无效时，\_\_block修饰符也能被用来避免循环引用。这是因为当block从栈复制到堆上时，若block使用的变量为附有\_\_block说明符的id类型的自动变量，不会被retain，反之如果没有\_\_block修饰符，则会被retain。这是因为在非ARC下对于栈上的_block对象, Block不会对其复制, 仅仅使用, 不会增加引用计数。

参考资料：

[https://blog.ibireme.com/2013/11/27/objc-block/#more-41448](https://blog.ibireme.com/2013/11/27/objc-block/#more-41448)
[http://www.jianshu.com/p/e23078c11518](http://www.jianshu.com/p/e23078c11518)
[https://www.jianshu.com/p/71f9114ba06f](https://www.jianshu.com/p/71f9114ba06f)

