# 拾壹：Runtime系列_(1)

> OC 的消息机制
> 
> 答： OC中的方法调用其实都是转成了`objc_msgSend`函数的调用，给receiver（方法调用者）发送了一条消息（`selector`方法名）消息机制有3大阶段,消息发送（当前类、父类中查找）、动态方法解析、消息转发。
> 
> 
> 什么是Runtime？平时项目中有用过么？
> 
> 答： OC的动态性就是由Runtime来支撑和实现的，Runtime是一套C语言的API，封装了很多动态性相关的函数
平时编写的OC代码，底层都是转换成了Runtime API进行调用。
>具体应用:
>
>1. 利用关联对象（AssociatedObject）给分类添加属性
>2. 遍历类的所有成员变量（修改textfield的占位文字颜色、字典转模型、自动归档解档）
>3. 交换方法实现
>4. 利用消息转发机制解决方法找不到的异常问题
>


## isa 指针

在[贰：对象的isa指针指向哪里](https://github.com/PhoenixiOSer/iOSLearning/blob/master/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/%E8%B4%B0%EF%BC%9A%E5%AF%B9%E8%B1%A1%E7%9A%84isa%E6%8C%87%E9%92%88%E6%8C%87%E5%90%91%E5%93%AA%E9%87%8C%EF%BC%9F.md)文章中提到过 isa 指针 在 arm64 架构之前只是简单 Class 的指针，但从64位架构之后 isa 的从指针类型变成 union 共用体/联合体类型：

```
struct objc_object {
private:
    isa_t isa;
	// other methds..
}

union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
#if defined(ISA_BITFIELD)
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };
#endif
};
```

**结构体和共用体的区别在于：结构体的各个成员会占用不同的内存，互相之间没有影响；而共用体的所有成员占用同一段内存，修改一个成员会影响其余所有成员。**

`typedef unsigned long uintptr_t;`

`uintptr_t` 被定义为`unsigned long `在 union 中的字节数最大（64位下占8个字节），所以类的所有信息其实是存放在 `uintptr_t  bits` 内存的每一位中。而`ISA_BITFIELD`是为了方便阅读而定义的结构体成员，没有实际含义。是让我们可以对照信息存放的位置，我们可以通过`bits`进行位运算获取`ISA_BITFIELD `中的所有信息。


### 位域

位域的声明：

```
struct
{
  [type] [member_name] : width ;
};
```
其中`type`、`member_name `分别代表类型、成员名称。而 `width` 表示位的个数。必须小于或等于指定类型的位数。

 `isa_t` 中的成员包括一个的结构体，包含了类的所有信息，arm64架构下的结构体：

```

struct {
	总共 64 位：
	//ISA_BITFIELD;  // defined in isa.h
	uintptr_t nonpointer      : 1;   // 占用1位，0 代表普通的指针，1 代表使用位域技术
	uintptr_t has_assoc         : 1; // 占用1位，是否有设置过关联对象
	uintptr_t has_cxx_dtor      : 1; // 占用1位，是否包含C++的析构函数
	uintptr_t shiftcls          : 33; // 占用33位，存储Class、Meta-Class对象的内存地址信息
	uintptr_t magic             : 6;  // 占用6位，用于在调试时分辨对象是否未完成初始化
	uintptr_t weakly_referenced : 1;  // 占用1位，是否有被弱引用指向过
	uintptr_t deallocating      : 1;  // 占用1为，对象是否正在释放
	uintptr_t has_sidetable_rc  : 1;   // 引用计数器是否过大无法存储在isa中
如果为1，引用计数会存储在SideTable中
	uintptr_t extra_rc          : 19 // 里面存储的值是引用计数器减1
};

/// 结构体中的信息位存放地址依次从最低位到最高位。
```


小知识：`shiftcls` 存放了类、元类的地址，**arm64架构** 时`define ISA_MASK        0x0000000ffffffff8ULL`,获取类和元类的地址会和`ISA_MASK `进行按位与（&）运算，所以最后三位肯定为 0,所以在打印地址时16进制的最后一位要么为 0、要么为8:

```
最后四位进行位运算：
 1111 // 某类对象地址的最后四位    0111 // 某类对象地址的最后四位   
&1000 // isa_mask				&1000 // isa_mask
----------						----------
 1000 // 8 						0000 // 0
```

## 方法及方法缓存

## 方法结构
[叁：class 和 meta-class的结构](https://github.com/PhoenixiOSer/iOSLearning/blob/master/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/%E5%8F%81%EF%BC%9Aclass%20%E5%92%8C%20meta-class%E7%9A%84%E7%BB%93%E6%9E%84.md)中提到过 class 的结构, 类中主要的信息存放在 `class_rw_t ` 中：

```
// 可读可写
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro; 

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;
    // other methods ..
 }

// 只读
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
    // other ...
}
```


其中`class_ro_t `是只读的，当类在编译的时候就确定了无法修改，而`class_rw_t `中的`const  class_ro_t *`也说明了无法我们无法对`class_ro_t `再进行修改，只能读取。

通过上述源码中，我们知道`class_rw_t `包括方法、属性、协议等。那么 OC 中方法又是怎样的结构呢？

要了解方法调用的流程，我们先来看下方法的存放结构，在`class_rw_t `中方式是存放在`method_array_t methods`中的，`methods `是指向指针的指针转换成数组来说就是一个二维数组, `method_array_t` 存放着 `method_list_t `,而 `method_list_t `存放着 `method_t `:

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/class_rw_t__methods.png?raw=true)

在 OC 中每个方法对应着一个 `method_t `：

```
struct method_t {
    SEL name; // 方法名称 
    const char *types; // 方法的 type encoding（主要是方法返回值及参数的信息）
    MethodListIMP imp; // 方法的实现地址
    // ... other 
};
```

`SEL`可以通过`@selector()`和`sel_registerName()`获得，`SEL`类似`char *`仅仅代表方法的名字，并且不同类中相同的方法名的SEL是全局唯一的。

`types`包含了函数返回值，参数编码的字符串。通过字符串拼接的方式将返回值和参数拼接成一个字符串，来代表函数返回值及参数。在[TypeEncodings官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100)有详细讲解，主要是通过 `@encode()`指令将类型转换成对应的`code`如下表：


![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/type_encoding.png?raw=true)


举个列子说明：`"i24@0:8i16f20"`

```
- (int)testWithAge:(int)age height:(float)height
{
    return 0;
}
  i    24    @    0    :    8    i    16    f    20
int         self        _cmd       int        float
// 参数的总占用空间为 8 + 8 + 4 + 4 = 24
// self 从第0位开始占据8位空间
```

`MethodListIMP`:可以认为就是IMP,存储的内容是函数地址。

`SEL`、`types`、`MethodListIMP`共同构成了一个方法的所有要素。

## 方法缓存

在调用方法的时候，我们会通过 `isa`指针和 `superclass` 指针一层一层向上遍历`method_array_t methods`直到找到方法或者一直到基类也没有找到方法后进行方法动态解析、消息转发、最后抛出`unrecognized selector sent to instance/Class`错误。

如`init`方法，子类如果都没有重写，那么就会一直寻找到 `NSObject` 的 `init`方法进行调用，如果没有进行方法缓存，每次子类调用`init`方法都会从子类开始一直寻找到  `NSObject` 。这会大大的增加了系统的开销，所以类针对方法查找后会进行缓存。

调用子类的 `init`方法的过程：

1. 首次子类的实例对象通过`isa`指针找到对应的子类，从子类本身的方法缓存 `cache` 中寻找，未找到通过 `bits` 定位到`class_rw_t`中`method_array_t`遍历寻找，如果找到方法把方法缓存到 `cahce`中。
2. 如果子类中未找到方法，通过 `superclass` 指针到父类中重复第一步的流程。直到根类，如果在父类中找到，同时将方法缓存到父类和调用方法的子类中。
3. 如果直到根类都未找到进行方法动态解析、消息转发。
4. 子类二次调用 `init`会在 `cache`找到直接调用，省去了前面的3个流程。大大提升系统效率。


在 OC 中 方法的缓存在`objc_class `中的 `cache_t` 结构体，`cache_t `包含一个 hash 表`_buckets`：

```
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // 方法缓存
    class_data_bits_t bits;    // 类的具体信息
    
   // other methods ... 
}

struct cache_t {
    struct bucket_t *_buckets;  // 缓存方法的hash表
    mask_t _mask; // (hash表的大小 - 1)
    mask_t _occupied; // hash 表被占用的大小(已缓存的方法数)
    // ohter methods...
}
	

struct bucket_t {
private:
	// 非__arm64__环境下 SEL 在 uintptr_t之前
    uintptr_t _imp;
    SEL _sel; 
	// other methods...
}
```

采用hash表来存放缓存的原因是：**无论多大的一个数据量，算法复杂度总是Ο(1)的。能够快速的找到方法**

hash表 `_buckets` 的 hash算法很简单直接是使用 `SEL` 按位与（&） `mask`,所以返回的索引肯定是在`mask`的范围内。

```

typedef uintptr_t SEL;
typedef unsigned long uintptr_t;
typedef uint32_t mask_t; // arm64、x86_64架构下

static inline mask_t cache_hash(SEL sel, mask_t mask) 
{
    return (mask_t)(uintptr_t)sel & mask;
}
```

我们都知道使用 hash 表的最大问题就是**碰撞**,两个不同`sel`最后计算得到的相同的索引（mask_t），那苹果又是如何解决这种碰撞的呢？


![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/hash_cache_t.png?raw=true)

通过`cache_hash `计算出的索引，如果通过索引获取的方法缓存`cache_t`中的`sel`非空或不等于参数s,即发生了hash碰撞，苹果解决碰撞的策略也很简单：

```
#if __arm__  ||  __x86_64__  ||  __i386__
// objc_msgSend has few registers available.
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return (i+1) & mask;
}
#elif __arm64__
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return i ? i-1 : mask;
}
#else
#error unknown architecture
#endif

```
直接是**非 arm64 架构**从当前位置向后(+1)继续寻找、**arm64架构**从当前位置向前(-1)，继续寻找正确的索引。
