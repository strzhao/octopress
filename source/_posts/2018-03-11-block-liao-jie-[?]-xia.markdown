---
layout: post
title: "谈谈 Objective-C 的 block 实现"
date: 2018-03-11 12:13:52 +0800
comments: true
categories: 
---
##我是前言
这篇文章是我自己最近在学习 block 后的一个简单总结。  
这篇总结你可以读到：   

- block 的本质是什么  
- block 如何捕获变量  
- block 生命周期管理

## block 的本质
block 是 Objective-C 对于闭包的实现，闭包用一句话来表示：带有自动变量的匿名函数。  
所以问题的核心就是 **Objective-C** 如何实现带有**自动变量**的**匿名函数**。  
先看看我们已经了解的，从 Objective-C 它自身来说就是 C + runtime (+ 编译器)，所以 block 也一定只能是 C 语言层面上的一个扩展。  
捕获变量和匿名函数，C 语言本身是不支持的，所以这里就需要编译器来帮忙了。  
我们一起来看看编译器都做了什么。  

我们通过执行
```
clang -rewrite-objc xxx.c
```
把下边含有 block 的代码转换成 C 语言代码来一探究竟。
```
int main(int argc, const char * argv[]) {

    void (^blk)(void) = ^{printf("Hello, World!\n");};
    blk();
    return 0;
}
```
转换后的代码核心部分如下：(同时把类型转换也去掉)
```
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackblock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    printf("Hello, World!\n");
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main(int argc, const char * argv[]) {

    void (*blk)(void) = &__main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA);
    blk->impl.FuncPtr(blk);
    return 0;
}
```
可以看到 block 通过在编译后就是普通的 C 语言代码了，我们逐个来分析下：
```
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    printf("Hello, World!\n");
}
```
匿名函数在根据所属的类以及在类中出现的顺序编译后就是一个普通符合某个规范命名的 C 函数了，其中的参数 `struct __main_block_impl_0 *__cself` 完全符合 Objective-C 对于 OC 方法处理，如：
```
- (void)method:(int)arg {
    NSLog(@"%@ %d", self, arg);
}
```
在编译成 C 语言后大概为这样：
```
void _I_Object_method(struct Object *self, SEL _cmd, int arg) {
    NSLog(@"%@ %d", self, arg);   
}
```
这也是为什么我们在 OC 方法中可以使用 self 指针。额外说一点 OC 的发送消息。
```
[obj method:1] // 会被编译成下边的形式
objc_msgSend(obj, sel_resigtName("method:"), 1); // 这里的 obj 就是传到函数中的 self 指针。
```
回到 block 中，我们继续去看看 struct __main_block_impl_0 这个 block 对象的结构。(我们把结构体拍扁)
```
struct __main_block_impl_0 {
    void *isa;  // 用于实现 block 对象的功能 后边说明
    int Flags;  // 后边说明
    int Reserved; // 保留字段
    void *FuncPtr; // 函数指针 在构造时会对应到定义好的匿名函数
    struct __main_block_desc_0* Desc; // 后边说明用途
}

struct __main_block_desc_0 {
  size_t reserved;  // 保留字段
  size_t block_size; // block 的大小
}
```
很多的字段在这个例子中暂时都没有使用到（isa 除外，这个也留到后边说）。  
我们先做一个记录，后边会再来解释这些字段:  

- isa 的作用是什么 ?
- Flags 的作用是什么 ?

然后我们来看看 block 的使用。
```
    void (*blk)(void) = &__main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA);
    blk->impl.FuncPtr(blk);
```
至此基本解决了匿名函数的问题，我们总结一下：匿名函数会被编译成一个 C 结构体对象和一个 C 函数，且 C 结构体拥有这个 C 函数指针。  
定义一个 block 就是初始化一个该 C 结构体的结构体指针，block 的调用就是调用该结构体指针拥有的 C 函数, 并且把自己作为参数传递进去。  

### block 如何捕获变量
C 函数中可能用到的变量有很多，全局静态变量和全局变量显然直接使用就好了，不需要特别处理。
我们来看看无法直接使用的，先从栈上的基本类型变量开始，继续使用 clang 编译一下代码。
```
int main(int argc, const char * argv[]) {

    int val = 3;
    void (^blk)(void) = ^{printf("Hello, World! val = %d", val);};
    blk();
    return 0;
}
```
删掉无关代码我们得到以下代码
```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int val;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _val, int flags=0) : val(_val) {
    impl.isa = &_NSConcreteStackblock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int val = __cself->val; // bound by copy
    printf("Hello, World! val = %d", val);
}

int main(int argc, const char * argv[]) {

    int val = 3;
    void (*blk)(void) = &__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, val);
    blk->impl.FuncPtr(blk);
    return 0;
}
```

从这里我们不难发现，参数 val 其实是作为 block 对应的结构体的一个成员变量在初始化时传递进去达到捕获的效果的，而使用的时候也只需要通过访问该结构体指针的成员变量即可。  
block 中操作的 val 和外边的 val 已经没有关系了，所以修改 block 中的 val 是没有办法修改外部的 val 的（编译器通过抛出错误告诉我们这样是行不通的），那么想要在 block 中修改 val 要怎么做呢，就要用到我们经常使用的 __block 修饰符。（还有虽然可以但是几乎不用的局部静态变量）

#### __block 修饰符
我们先合理的猜测一下 ``__block`` 修饰符会做什么，先前 block 通过构造结构体捕获了匿名函数和变量，达到可以随意使用匿名函数和变量的目的，现在会怎么做呢？  
想好了，那我们继续来看 ``__block`` 修饰符都做了什么。
```
int main(int argc, const char * argv[]) {

    __block int val = 3;
    void (^blk)(void) = ^{printf("Hello, World! val = %d", val);};
    blk();
    return 0;
}
```
得到以下代码
```
struct __block_byref_val_0 {
  void *__isa;
  __block_byref_val_0 *__forwarding;
  int __flags;
  int __size;
  int val;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __block_byref_val_0 *val; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __block_byref_val_0 *_val, int flags=0) : val(_val->__forwarding) {
    impl.isa = &_NSConcreteStackblock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __block_byref_val_0 *val = __cself->val; // bound by ref
  printf("Hello, World! val = %d", (val->__forwarding->val));
}
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {
    _block_object_assign((void*)&dst->val, (void*)src->val, 8);
}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {
    _block_object_dispose((void*)src->val, 8);
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

int main(int argc, const char * argv[]) {

    __block_byref_val_0 val = {(void*)0,(__block_byref_val_0 *)&val, 0, sizeof(__block_byref_val_0), 3};
    void (*blk)(void) = &__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__block_byref_val_0 *)&val, 570425344);
    blk->impl.FuncPtr(blk);
    return 0;
}
```

我们可以看到变量 val 已经变成了结构体 ``__block_byref_val_0``, 之前的 block 对象 ``__main_block_impl_0`` 也拥有了 ``__block_byref_val_0`` 结构体实例指针，修改 val 就变成了修改结构体指针指向的结构体实例的变量 val 的值。  
可能比之前我们想象的要复杂一些，但从本质来说就是通过直接传值修改为传指针，这和我们之前的猜测应该是差不多的。  
但是似乎事情也变得复杂了很多，我们先罗列以下，后边一一解释。  

- ``__block_byref_val_0`` 中 ``__forwarding`` 指针的意义是什么？
- ``__main_block_desc_0`` 中 ``copy`` 和 ``dispose`` 函数的意义是什么？

加上之前留下的问题，都和 block 的生命周期有关

- isa 的作用是什么 ?
- Flags 的作用是什么 ?

### block 的生命周期管理

#### isa 的作用是什么
我们一直在说 block 对象，却也一直没有谈到 block 这个对象的生命周期要如何管理，所以我们先来解决 block 自身的生命周期管理问题。  
而其中的关键就是 isa 指针。  
~~isa 在 OC 对象设计中的作用不是本次的目的，暂时忽略不表~~  
我们来看看 block 的 isa 的几个类型：  

- _NSConcreteGlobalblock 程序的数据区域
- _NSConcreteStackblock 栈上 出了作用域就释放
- _NSConcreteMallocblock 堆上 引用计数为 0 时释放

而栈上的 block 可以复制到堆上来延长生命周期。  
在下边这些情况下编译器可以正确的处理复制：  

- 调用 `copy` 方法
- block 作为函数的返回值
- 将 block 赋值为 ``__strong`` 修饰的 id 类型 或者 block 类型的成员变量
- 在方法名中含有 ``usingBlock`` 的 Cocoa 框架方法 和 GCD 的 的方法

例如 block 作为函数返回的情况：
```
float ^(float x) doubel(float x) {
    return ^(float x) {
        return x * 2;
    };
}

// 编译后

float ^(float x) doubel(float x) {

    blk tmp = &__func_block_impl_0(__func_block_func_0, &__func_block_desc_0_DATA, x);

    tmp = objc_retainBlock(tmp); // 从栈上复制到堆上

    return objc_autoreleaseReturnValue(tmp); // 加到 autoreleasepool 中
}
```
这也是为什么 block 在超过变量作用域后还可以继续使用的原因。  

~~但是在某些情况下系统是无法高效的做出判断的，那么就需要我们自己来调用 `copy` 把 block 复制到堆上，例如 `[NSArray initWithObjects:]` 方法，如果我们不手动的执行 `copy` 当获取数组中的对象并且执行时是会崩溃的。大家可以自行尝试。~~ MRC 情况下才会出现, ARC 不会。

#### __forwarding 的意义





















