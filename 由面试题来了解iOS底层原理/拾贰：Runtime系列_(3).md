# 拾贰：Runtime系列_(3)

来看一些 runtime 综合的面试题:

### class相关面试题,下面代码会输出什么？

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/super.png?raw=true)

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/super_console.png?raw=true)

[self class] 和 [self superclass] 的输出应该都没啥意外， 但是很多人可能会认为 super 调用 class、 superclass 输出应该是 `Person` 和 `NSObject`。为什么 super 输出还是和 self 的一样？这其实和 super 调用方法的过程有关，我们先去掉 `init` 来看下 super 的调用，使用
`xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc -fobjc-arc -fobjc-runtime=ios-13.0.0 Student.m`

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/super_student.png?raw=true)

`test` 方法编译生成的 c/c++ 代码：

```
static void _I_Student_test(Student * self, SEL _cmd) {
    ((void (*)(__rw_objc_super *, SEL))(void *)objc_msgSendSuper)((__rw_objc_super){(id)self, (id)class_getSuperclass(objc_getClass("Student"))}, sel_registerName("test"));
}

// 简化去除类型转换的
objc_msgSendSuper({self, class_getSuperclass(objc_getClass("Student"))},
                   sel_registerName("test"));

// objc_msgSendSuper(struct objc_super * _Nonnull super, SEL  _Nonnull op)
```

`objc_msgSendSuper` 函数的第一个参数是一个`objc_super `结构体指针,第二个是 SEL.

```
struct objc_super {
 id receiver;
 Class super_class;
};
```

跟我们前面简化去除类型转换的 test 方法调用比对，其中 receiver 是 self， super_class 是`class_getSuperclass(objc_getClass("Student")`。

在这里需要注意的是，虽然 `test` 通过 clang 编译出来是调用了`objc_msgSendSuper `函数但是真正调用的确是`objc_msgSendSuper2`函数。

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/super_objc_msgsendsuper2.png?raw=true)

在 objc4 源码中的` objc-msg-arm.s `文件下可以找到`super_objc_msgsendsuper2 `的汇编实现（不太会汇编，直接看代码的注释）：

```	
/********************************************************************
 * id objc_msgSendSuper(struct objc_super *super, SEL op, ...)
 *
 * struct objc_super {
 *     id receiver;
 *     Class cls;	// the class to search
 * }
 ********************************************************************/
 
/********************************************************************
 * id objc_msgSendSuper2(struct objc_super *super, SEL op, ...)
 *
 * struct objc_super {
 *     id receiver;
 *     Class cls;	// SUBCLASS of the class to search
 * }
 ********************************************************************/

```

`super_objc_msgsendsuper`、`super_objc_msgsendsuper2`两者间的区别在于的第二个参数一个是父类、一个是子类，相当于：

```
struct objc_super {
 id receiver;
 Class super_class;
};

struct objc_super2 {
    id receiver;
    Class current_class;
};
```
相比较于`objc_super `直接传递父类，`objc_super2`是传递了当前类`current_class `,在`super_objc_msgsendsuper2 `中在通过`current_class->superclass`获取父类。

回到题目，在 Student 中虽然使用 super 调用 test,但真正的消息接受者(receiver)还是 self (Student 对象)。而第二个参数只是决定了从父类开始寻找方法。

在 `init`方法中 [self class] 和 [super class] 的 receiver 都是 Studnet，区别就在于从哪个类开始方法寻找，self 从类本身开始，super 从父类开始寻找。而 Person 和 Student 都没有实现 class 方法，最终会找到 NSObject 类中的 `-class` 对象方法：

```
- (Class)class {  
    return object_getClass(self);
}

//  类方法返回自身
+ (Class)class {
    return self;
}
```

`object_getClass(self)`其中 `self` 是 `Student` 的实例对象，所以输出都是 `Student`。

同样的道理 `self`、 `super` 再调用 `superclass ` 方法时， `receiver` 还是 self（Student）寻找 superclass 会一直到 NSObject 类中的 `-superclass` 的实现：

```
- (Class)superclass {
    return [self class]->superclass;  // Student->superclass
}

+ (Class)superclass {
    return self->superclass;
}
```

所以返回的都是 Person。


### isKindOfClass 、isMemberOfClass 面试题 下面代码会输出什么：

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/iskindof.png?raw=true)

我们可以在 objc4 的 NSObject.mm 源码中能够找到 `isKindOfClass `,`isMemberOfClass`的实现：

```
+ (BOOL)isMemberOfClass:(Class)cls {
    return object_getClass((id)self) == cls;
}

+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = object_getClass((id)self); tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}

- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}
```

面试题中，是类对象调用 `isKindOfClass `,`isMemberOfClass` 方法，所以我们只关注类方法：

-  `+isMemberOfClass` 是根据 `self` 的元类是否和传进来的 cls 类是否相等。
-  `+isKindOfClass `除了判断 `self` 的元类和传进来的 cls 类是否相等外，还会和 cls 的父类进行比对直到父类为 `nil`。

能够理解以上两条信息的话，上面的答案也就一目了然了。

- `[[NSObject class] isMemberOfClass:[NSObject class]];`：在前文我们知道 `+class` 返回的是类对象本身。代码就相当于 `[NSObject isMemberOfClass:NSObject];`判断 NSObject元类和NSObject类对象是否相等，答案明显输出 `NO`
- `[[NSObject class] isKindOfClass:[NSObject class]];`：代码就等价`[NSObject isKindOfClass:NSObject];` NSObject元类 和 NSObject类对象肯定不等，但是这里有个细节需要注意的是，NSObject元类是继承于NSObject类（下图红框选中的线条），所以当向父类中进行比对，NSObject元类的父类就是NSObject类，输出YES。

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/isa_inherit.png?raw=true)

- 同理 `Person` 的 `isMemberOfClass `和`isKindOfClass `方法判断也是一样的。

### super相关面试题

下面代码能否编译成功？如果能，会输出什么？

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/question_print.png?raw=true)

答案是肯定的，代码能够编译成功，但是输出的是`<ViewController: 0x7fc0afc194f0>`,打印了控制器对象。

#### 为什么能够编译成功？

根据之前所学的知识，我们知道 oc 中调用方法都是通过消息发送，通过 isa 指针到类对象中寻找对象方法:

```
    Person *p = [[Person alloc] init];
    [p print]; 	
    //输出：(null)
```

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/obj_person_cls.png?raw=true)

上图中我们可以看到`obj`指向`cls`再指向 `Person` 的类对象和 p 指向`Person`的对象， `Person`结构体中的 `isa` 再指向的 Person 类对象的路径完全吻合。而且获取 isa 其实就是读取内存的前 8 个字节，所以最后都指向了`Person 类对象`。

### 输出为什么是控制器对象

上面我们知道了编译能够通过的原因，那么输出的结果又是为什么是控制器对象呢？


在这之前我们先来看另一个奇怪的现象：

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/super_helloworld.png?raw=true)

只要在定义 cls 之前在添加一个 `test` 字符串，那么输出的结果竟然变成了`hello world`。

想要了解为什么会有这种奇怪的输出，需要我们对量内存的分配有一定了解，首先要知道在 OC 中局部变量在栈上的内存是从高地址到地址的顺序分配的：

```
    long long a = 1;
    long long b = 2;
    long long c = 3;
    
    NSLog(@"%p,%p,%p",&a,&b,&c);
    /// 0x7ffeeac2d208,0x7ffeeac2d200,0x7ffeeac2d1f8
```

可以看到 a 存放在了高地址（208），接下来是b（200）、c(1f8) 依次存放。上面代码的局部变量栈空间分配：

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/super_name.png?raw=true)


```
@implementation Person

- (void)print {
    NSLog(@"%@",self.name);
}
@end
```

`self.name 等价 self->name` 就是访问 Person 的 name 成员变量。从 Person 对象的结构体首地址开始跳过 8 个字节读取另外 8 个字节的内存。

```
    NSString *name = @"hello world";
    
    id cls = [Person class];
    void *obj = &cls;
    [(__bridge id)obj print];
```

而 `[obj print]` 就是向 `obj` 发送 `print`消息，通过 cls 指针 类似 isa 指针的作用 指向了 Person 类中存放的 print 方法，但是此时 print 中的 self 是 `obj`, 读取 name 成员变量等价于跳过 8 个字节（跳过了cls指针）读取内存，读取到局部变量 test，输出`hello world`（可以结合上图的内存分布来一起理解）。


在了解了内存分配和访问的机制后，去掉 `test` 的局部变量，我们再来看看:

```
- (void)viewDidLoad {
    [super viewDidLoad];
        
    id cls = [Person class];
    void *obj = &cls;
    [(__bridge id)obj print];
}    
```

输出就变成 `<ViewController: 0x7fc0afc194f0>`。根据我们前面的了解，读取的肯定 cls 的前8个字节。但是在代码中并没有出现定义其他的对象。其实问题出在`[super viewDidLoad];`

在前文第一个面试题中我们知道了 super 调用方法真正的函数调用：

```
// objc_msgSendSuper2(struct objc_super * _Nonnull super, SEL  _Nonnull op)
函数的第一个参数是一个`objc_super2`结构体指针

struct objc_super2 {
 id receiver;
 Class super_class;
};
```

会有一个中间结构体变量`objc_super2`,那么`[super viewDidLoad];`可以看做：

```
struct objc_super2 = {
	self(ViewController 对象),
	[ViewController Class]
};
objc_msgSendSuper2(objc_super2,@selector(viewDidLoad));
```

在 cls 之前会有一`objc_super2`结构体，我们会读取到`objc_super2`时读取前 8 个字节就是 self(ViewController 对象)，输出了`<ViewController: 0x7fc0afc194f0>`。

很无厘头的一个题目，一步一步分析过来才慢慢的懂一点。无F可说。

