# ARC 到底到底是怎么实现

日常写代码离不开ARC，我们天天写Strong ，Weak 这些关键字，也都知道ARC，MRC，但是这两种内存管理之间到底是怎么实现的，以及有什么关系 我想需要整理明确一下。

内存管理 -- 内存种类 管理的主要是那一部分的内存？

## 内存的种类

首先我们的进程在启动的时候操作系统会分配一段内存，除了我们熟知的栈区（stack），堆区（heap）外还有全局区/静态区（static），常量区，代码区。 先说不常用的

### 代码区
* 存放我们app的代码。
* 特点是为了保护代码不被修改 该区域是不允许写操作的。
* 程序结束后自动释放。

### 全局区/静态区
* 主要存放初始化/未初始化的 全局变量以及静态变量。
* 初始化的全局变量以及静态变量在一起 存放在已初始化数据段（.data）,未初始化的存放在未初始化数据段（.bss）
* 同代码区一样，也是程序结束后自动释放


### 栈区

* 栈区的大小在线程创建的时候就已经固定了，比如iOS主线程 栈区大小为512k，我们可以通过NSThead的stackSize方法获取或者设置，但是需要再线程启动之前设置好栈区的大小，所以说主线程的栈大小没办法直接控制。

* 栈区是一种“先进后出”的存储结构，我们常用的方法调用就是一个进栈的过程，方法执行完后出栈，保证了代码的顺序执行，方法中的参数变量（如果是引用类型栈中持有的是指针）也就存放在了栈区，如果方法一直进栈没有出栈的过程，因为我们的栈是有空间限制的所以就会出现“栈溢出”。

* 栈区的内存不需要我们来管理，由编辑器自动申请以及释放。

### 堆区

* 堆区内存是由 alloc 方法申请的内存，所以堆内存不一定是连续的，我们调用alloc方法，操作系统会为我们申请一块我们需要的内存大小，具体操作系统如何管理内存申请的 可以看这篇文章 [Linux堆内存管理深入分析](https://introspelliam.github.io/2017/09/10/Linux%E5%A0%86%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%B7%B1%E5%85%A5%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%8A%EF%BC%89/)，操作系统分配内存后并不会管理这块内存的释放问题，所以我们需要管理的内存主要来自堆内存。

* C/C++中申请的内存需要手动调用free方法才可以释放，在iOS中我们主要有ARC，MRC，两种内存管理机制，这两种的主要差别以及实现原理分别是什么，这就是我想搞清楚的。


## iOS内存管理的实现
oc的内存管理是通过引用计数的方式实现的，标志位retainCount 当retainCount == 0 的时候就将该对象释放，那么先抛出问题，retainCount存在哪？ retain ，release ，autorelease到底干了什么，以及在ARC下 strong ，weak 到底做了什么，Arc到引用计数之间是怎么过度的？

我们可以下载OCJC的源码，来查看一个oc对象内部的实现
下载地址
https://opensource.apple.com/source/objc4/objc4-756.2/

在解决问题之前先认识一下NSObject 内部的实现，直接从下载的源码中就可以看到NSObject

``` c
@interface NSObject <NSObject> {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wobjc-interface-ivars"
    Class isa  OBJC_ISA_AVAILABILITY;
#pragma clang diagnostic pop
}

typedef struct objc_class *Class;

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
}

struct objc_object {
    isa_t isa;
}

union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
};

```
只有一个 Class 类型的 isa 成员变量，那么Class又是 objc_class 这个结构体,objc_class 又继承自 objc_object 结构体所以 Class 的内部成员变量可以清楚了
* isa_t isa
* Class superClass
* cache_t cache
* class_data_bits_t bits

那么就会发现 ，一个NSObject 对象 内部有一个Class 类型的 isa 成员变量，而这个 Class 内部又有一个isa 的成员变量，这两个isa到底是干什么的，那么就需要理解实例对象，类对象，元类对象。
实例对象很简单 
```
NSObject *obj = [[NSObject alloc] init];
Class objClass = [obj Class];
Class metaClass = object_getClass(objClass)
```

其中obj就是我们常用的实例对象,
objClass 就是类对象
metaClass 就是元类对象

其中实例对象中存放的是我们的成员变量的值。
类对象存放的是实例方法。
元类对象存放的对象方法。

简单了解对象有实例对象，类对象，元类对象后，NSObject中的isa 成员变量其实就是用来指向类对象的标识，类对象中的isa指针就是指向元类对象。



### retainCount
