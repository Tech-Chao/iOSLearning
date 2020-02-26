# 玖：Block 原理面试（1）
>
>- block的原理是怎样的？本质是什么？
>
> 答：Block 的本质是一个封装了函数及其调用环境的 Objective-C 对象。原理详细见「Block 使用及结构」
> 
> - block的属性修饰词为什么是copy？使用block有哪些使用注意？
> 
> 答： MRC 下 block 如果没有 copy 到堆上,值捕获不会对外部变量引用。 虽然 ARC 环境 strong 也可以修饰 Block，那是因为编译器会对 strong 修饰的 block 也会进行一次 copy 操作。为什么用 copy 修饰算是历史习惯问题，推荐不管 ARC、MRC 使用 copy 修饰 。使用注意：循环引用问题
> 



`Tip:本文中以下代码均为 ARC 环境，除非特别注明 MRC。`

## Block 使用及结构

来看一段简单的 Block 的代码：

```
// main.m

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        void(^block)(void) =  ^{
            NSLog(@"hello world");
        };        
        block();
    }
    return 0;
}
```

然后通过 ` xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m ` 查看编译后的 C++ 代码。

```
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        void(*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));

        ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
    }
    return 0;
}
```

可以看到 block 在编译之后转换成了`__main_block_impl_0 `结构体，结构体的包含的成员如下：

```
struct __main_block_impl_0 {
  
  // 相当于copy 了整个struct __block_impl impl
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
  // 相当于copy 了整个struct __block_impl impl
  	
  // Des 指针（描述 block 的大小	）
  struct __main_block_desc_0* Desc;  
  // 构造函数
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/block_struct.png?raw=true)

`__main_block_impl_0 ` 结构体和对象结构类似，首个成员是 isa 指针，指向类对象，由此可以推断 block 可能也是 OC 对象(在下文「Block 类型」中详细说明）。


此外 `__main_block_impl_0` 的 `FuncPtr` 函数指针指向了封装 block 代码块的函数 `__main_block_func_0 `:

```
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
            NSLog((NSString *)&__NSConstantStringImpl__var_folders_s5_1zc18cl97tn280nn_vywbl1m0000gp_T_main_23bff8_mi_0);
}     
```

一切就绪之后在`main`函数中开始执行block。

```
int main(int argc, const char * argv[]) {
    	// __AtAutoreleasePool 后面的文章在做讲解
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
		 // 去除强制转换后简化的代码
        void(*block)(void) = &__main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA));
		block->FuncPtr(block);
    }
    return 0;
}
```

block 结构体小结：
![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/block_impl.png?raw=true)

其中`copy` 和 `dispose` 两个函数下文「对象类型的值捕获」会提到。


## Block 值捕获（基本数据类型）


### 简单的带参数 Block （不会进行值捕获）

```
	void(^block)(int,int) =  ^(int a, int b){
		NSLog(@"a = %d, b = %d",a,b);
	};
	block(20,20);
```

带参数的 block, 在编译之后`__main_block_impl_0 `、`__main_block_desc_0 `结构并未发生变化。只有`__main_block_func_0`在定义和使用中新增了连个 a, b 参数。这种 block 并不涉及到值捕获。

```
static void __main_block_func_0(struct __main_block_impl_0 *__cself, int a, int b) {
	NSLog((NSString *)&__NSConstantStringImpl__var_folders_s5_1zc18cl97tn280nn_vywbl1m0000gp_T_main_aec4c2_mi_0,a,b);
}

void(*block)(void) = &__main_block_impl_0(__main_block_func_0,&__main_block_desc_0_DATA));
block->FuncPtr(block,20,20);

```

### 局部变量捕获

#### 捕获auto变量

简单的 auto 变量地址捕获:

```
	  // 局部变量默认 auto 修饰
     int age = 10;  // 相当于 auto int age = 10;
     void(^block)(void) =  ^{
         NSLog(@"age is %d",age);
     };
     age = 20;
     block();
// 输出 
age is 10
```

如果在 block 中访问了 auto 变量， block 的结构体会发生什么变化呢：

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int age;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _age, int flags=0) : age(_age) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

在上面的`__main_block_impl_0 `结构体中新增加一个 `int age;`成员。`__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _age, int flags=0): age(_age) `构造方法也有了一个 `_age`参数 函数将 `_age` 赋值给了结构体的 age 成员属于值传递。


```
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int age = __cself->age; // bound by copy

  NSLog((NSString *)&__NSConstantStringImpl__var_folders_s5_1zc18cl97tn280nn_vywbl1m0000gp_T_main_232207_mi_0,age);
}
```

在执行 block 中的代码块函数时，`__main_block_impl_0 `中的 age 是值传递与局部变量 age 无关，所以即使外部的 age 变量修改了值。也是不会影响 block 中早已捕获的 age。

#### 捕获static变量

block 捕获静态变量

```   
    static int age = 10;
    void(^block)(void) =  ^{
        NSLog(@"age is %d",age);
    };
    age = 20;
    block();
    
// 输出 
age is 20
```

如果 block 捕获的是静态变量， block 的结构体又会发生什么变化？经过 clang 编译之后：

```
      static int age = 10;
        void(*block)(void) = &__main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA, &age));
        age = 20;
        block->FuncPtr(block);

```

和之前 auto 变量比较，static 传递的参数是 `age`的地址属于地址传递，`__main_block_impl_0` 的成员 `int *age` 存放的是 age 的地址，访问的是同一块内存，所以 age 在外部更改之后，block 中的 age 指向的值也会变动。

```

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int *age;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_age, int flags=0) : age(_age) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

局部变量捕获 auto 和 static 的区别

- auto 变量会在作用域之后销毁，所以 block 会将 age 进行值传递，并存放` __main_block_impl_0 `成员 age 中，用于以后可以随时访问。
-  static 的变量在初始化后会一直存放内存中，所以我们可以通过地址直接访问，不用担心变量作用域的问题，block 结构体的构造方法传递的是静态变量 age 的地址。


### 全局变量


```
static int age = 10;
int height = 30;
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        void(^block)(void) =  ^{
            NSLog(@"age is %d，height is %d",age,height);
        };
        age = 20;
        height = 40;
        block();
    }
    return 0;
}
```

经过 clang 编译之后：

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

发现 `__main_block_impl_0 `结构体中没有任何的值捕获的成员变量，是因为当 block 中的代码块需要访问全局变量时，可以直接访问， block 没有必要在进行值捕获。

```
// 直接访问全局变量 和 全局静态变量
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {

NSLog((NSString *)&__NSConstantStringImpl__var_folders_s5_1zc18cl97tn280nn_vywbl1m0000gp_T_main_d25360_mi_0,age,height);
}
```

## Block 类型

在前面的 Block 结构体中都存在一个 isa 指针，且在构造函数的时候赋值 `&_NSConcreteStackBlock`。所以可以猜测认为 block 其实也是对象的一种，
尝试对 block 调用 class 方法来看看会有什么输出：

```
	Class cls = [block class];
	while (cls) {
		NSLog(@"%@",cls);
		cls = [cls superclass];
	}

// 依次输出：
__NSGlobalBlock__
__NSGlobalBlock
NSBlock
NSObject

```
可以看出来 block 确实是对象且主要的 block 类型(都是继承自`NSBlock`)有以下三种：

- `__NSGlobalBlock__`（ _NSConcreteGlobalBlock ）存放在 **数据段** 中
- `__NSStackBlock__`（ _NSConcreteStackBlock ） 存放在 **栈** 中
- `__NSMallocBlock__`（ _NSConcreteMallocBlock ）存放在 **堆** 中

block 是属于哪一种类型总结下来可以用下面的图片表示：

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/block_type.png?raw=true)

```
		// ARC 下赋值给 __Strong（默认）的 变量时会自动调用 copy方法，将 block copy到堆上，无法准确查看 block 类型
		// 下面代码为 MRC 环境

		// __NSGlobalBlock__
        void(^block1)(void) =  ^{
            NSLog(@"hello world");
        };
        
        // __NSStackBlock__
        void(^block2)(void) =  ^{
            NSLog(@"hello age:%d",age);
        };
        // __NSMallocBlock__
		void(^block3)(void) = [block2 copy];
        NSLog(@"block1:%@,block2:%@,block3:%@",block1,block2,block3);
       // release 省略下...
       
        // 输出：
        block1:<__NSGlobalBlock__: 0x1000010a8>,
        block2:<__NSStackBlock__: 0x7ffeefbff480>,
        block3:<__NSMallocBlock__: 0x100638080>
```

补充： ARC 环境下下列操作会自动 block 进行 copy 操作：

- block 作为方法的返回值- 将 block 赋值给 __strong 指针时- block 作为Cocoa API中方法名含有usingBlock的方法参数时- block 作为GCD API的方法参数时

## Block 值捕获（对象类型）

前面提到的值捕获都是基本数据类型，如果在 block 捕获的值是对象类型的话， block的结构体又会发生什么变化呢？

```
	Person *p = [Person new];
	p.name = @"hello block!";
	void(^block)(void) = ^{
		NSLog(@"--- %@",p.name);
	};
	block();
```

将上面的代码 `clang` 编译之后:

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/block_object.png?raw=true)

对比之前捕获的普通 auto 变量，可以在图中看到 block 捕获的对象变量 `Person *p`时在 `desc`中新增了两个函数的指针:

```
 void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
 void (*dispose)(struct __main_block_impl_0*);
```
在 block 执行构造函数时，会对赋值两个函数的地址。

`_Block_object_assign`函数会在 block 进行一次 copy 操作的时候被调用。

```
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {
	_Block_object_assign((void*)&dst->p, (void*)src->p, 3/*BLOCK_FIELD_IS_OBJECT*/);
}
```

`_Block_object_assign`函数会根据 auto 变量的修饰符（`__strong（默认）`、`__weak`、`__unsafe_unretained`）做出相应的操作，block 结构体中的 `Person *p` 对外部的 auto 变量形成强引用（strong）或者弱引用（weak）。

如果block从堆上移除时，会调用 block 内部的`_Block_object_dispose`函数。

```
static void __main_block_dispose_0(struct __main_block_impl_0*src) {
	_Block_object_dispose((void*)src->p, 3/*BLOCK_FIELD_IS_OBJECT*/);
}
```

`_Block_object_dispose`函数会对结构体中的 `Person *p` 进行 release 操作。


```
enum {
    /* See function implementation for a more complete description of these fields and combinations */
    BLOCK_FIELD_IS_OBJECT   =  3,  /* id, NSObject, __attribute__((NSObject)), block, ... */
    BLOCK_FIELD_IS_BLOCK    =  7,  /* a block variable */
    BLOCK_FIELD_IS_BYREF    =  8,  /* the on stack structure holding the __block variable */
    BLOCK_FIELD_IS_WEAK     = 16,  /* declared __weak, only used in byref copy helpers */
    BLOCK_BYREF_CALLER      = 128  /* called from __block (byref) copy/dispose support routines. */
};
```

### 补充：

- 如果 block 如果在栈上，自身的生命周期都不确定，所以无法对外部变量进行引用。当 block 是`__NSStackBlock__ `类型是不会对 auto 变量进行强引用。

- `__weak` 的作用：

```
   __weak Person *weakPerson = p;
   void(^block)(void) = ^{
     NSLog(@"--- %@",weakPerson.name);
   };
   block();   
```

使用`xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc -fobjc-arc -fobjc-runtime=ios-13.0.0 main.m`clang 编译后``__main_block_impl_0``区别在于 weakPerson是弱引用:

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  Person *__weak weakPerson;
}
```

- block 的属性修饰

在 MRC 环境下:

```
@property (copy, nonatomic) void (^block)(void);```
在 ARC 环境下block属性的可以用 strong、copy 修饰，ARC 环境下会默认给赋值 strong 的block进行一次 copy 操作。但一般推荐使用 copy 修饰。算是代码习惯。

```@property (strong, nonatomic) void (^block)(void);@property (copy, nonatomic) void (^block)(void);
```
