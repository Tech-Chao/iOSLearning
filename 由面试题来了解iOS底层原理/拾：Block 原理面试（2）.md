# 拾：Block 原理面试（2）

> - __block的作用是什么？有什么使用注意点？
> 
> 答：`__block`可以用于解决 block 内部无法修改auto变量值，`__block`不能也没必要修饰全局变量、静态变量（static）详见下文。
> 
> - 使用 block 的注意事项?如何解决循环引用?
> 
> 答： block 使用主要关注内存是否泄漏，block 容易造成循环引用，解决循环引用主要有两种：
>
>  1. 使用 `__weak`（推荐）、`__unsafe_retained`(不推荐)修饰被 block 的捕获的变量。
> 2. 在block执行的代码块中，将捕获的变量重置为 nil，缺点是必须执行完 block 块才会解决循环引用。



## __block 的原理

```	
	// 编译报错
	int  age = 10;
	void(^block)(void) = ^ {
		age = 20;
	};

```

上述代码在 Xcode 中编译的时候就会报错，block 是无法直接修改外部 auto 变量。但static 的变量和全局变量也可以直接在 block 做出修改。因为 static 变量传递给 block 的是变量的地址，全局变量则是一直存在内存中，block 都可以访问并修改。详见[玖：Block 面试（1）-值捕获](https://github.com/PhoenixiOSer/iOSLearning/blob/master/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/%E7%8E%96%EF%BC%9ABlock%20%E5%8E%9F%E7%90%86%E9%9D%A2%E8%AF%95%EF%BC%881%EF%BC%89.md)。


### __block作用

```
	__block int  age = 10;
	__block NSObject *objc = [[NSObject alloc] init];
	
	void(^block)(void) = ^ {
		objc = nil;
		age = 20;
	};
```

通过 clang 命令`1`编译后：

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/__block.png?raw=true)

简化代码后：
![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/__block_simple.png?raw=true)

```
struct __Block_byref_age_0 {
  void *__isa;
__Block_byref_age_0 *__forwarding;
 int __flags;
 int __size;
 int age;
};
struct __Block_byref_objc_1 {
  void *__isa;
__Block_byref_objc_1 *__forwarding;
 int __flags;
 int __size;
 void (*__Block_byref_id_object_copy)(void*, void*);
 void (*__Block_byref_id_object_dispose)(void*);
 NSObject *__strong objc;
};
```

可以发现添加`__block`之后的变量转换成了 `__Block_byref_age_0`、`__Block_byref_objc_1`的结构体的类型，且对象类型生成的`__Block_byref_objc_1 `会比基本数据类型生成的结构体多两个内存管理的函数指针（`__Block_byref_id_object_copy`、`__Block_byref_id_object_dispose`）：

```
/**
	__block 修饰外部变量编译生成的 struct 对外部变量内存管理的两个方法
	不要和 block 内部中捕获对象变量生成的结构体时的内存管理方法搞混。
*/
/// __block 修饰外部变量生成的 struct 生成的内存管理方法
static void __Block_byref_id_object_copy_131(void *dst, void *src) {
 _Block_object_assign((char*)dst + 40, *(void * *) ((char*)src + 40), 131);
}
static void __Block_byref_id_object_dispose_131(void *src) {
 _Block_object_dispose(*(void * *) ((char*)src + 40), 131);
}

/// block 内部对捕获对象变量生成的结构体内存管理的两个方法
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->objc, (void*)src->objc, 8/*BLOCK_FIELD_IS_BYREF*/);_Block_object_assign((void*)&dst->age, (void*)src->age, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->objc, 8/*BLOCK_FIELD_IS_BYREF*/);_Block_object_dispose((void*)src->age, 8/*BLOCK_FIELD_IS_BYREF*/);}

```


`__block` 修饰的变量编译后的结构体地址传递给 block 被强引用着（默认），而每个`__block`修饰符产生的结构体内部又有一个引用着外部变量的成员（如上文中`__Block_byref_age_0 `的 age，`__Block_byref_objc_1 `的objc），对外部变量的引用是 `strong`还是 `weak` 则依赖值捕获时外部变量自身的修饰属性。

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/__block_reference.png?raw=true)


另外`__block` 修饰的变量编译后的结构体包含一个`__forwarding `指向结构体本身的指针。添加 `__block` 修饰之后的变量无论是在 block 内部还是外部访问、修改变量的值都会通过结构体内部`__forwarding `指针，如：`age.__forwarding->age = 30;` `__forwarding` 指针主要功能是用来确保无论从 block 外部还是内部都能够访问到正确的变量地址。


### `__forwarding` 指针

前面提到过 `__forwarding` 指针主要功能是用来确保能够访问到正确的变量地址，那么它是如何确保的呢？

以上文为例`__block int  age = 10;`在未被 block copy到堆上的时候`__forwarding` 指针指向的是栈上的结构体：

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/__forwarding_stack.png?raw=true)

当 block 被 copy 到堆上的时候,栈上变量的`__forwarding` 被指向了堆中的结构体地址，以后无论从栈上还是堆上访问结构体都会访问到堆：

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/__forwarding_%20heap.png?raw=true)



## Block对变量的内存管理总结

- block 在栈上的时候，不会强引用外部的任何变量
- block 从栈到堆上的时候，会有一次 copy 操作，在 copy 操作的时`__main_block_copy_0 `函数会根据捕获外部变量的 `strong`、`weak`修饰，来直接对强/弱引用外部变量。
- 如果捕获变量是 `__block` 修饰的，copy 到堆上的 block 会将变量转换成的结构体 copy 到堆上同时生成强引用，变量转换成的结构体自身对外部变量的强弱引用则是根据捕获变量时变量自身的强弱修饰符决定。
- 如果堆上的 block 被销毁了，`__main_block_dispose_0 `会对 block 引用的变量进行 release 操作。
