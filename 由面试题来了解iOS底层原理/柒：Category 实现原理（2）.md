# 柒：Category 实现原理（2）

> - Category中有load方法吗？load方法是什么时候调用的？load 方法能继承吗？
> 
> 答：有load方法，load方法在runtime加载类、分类的时候调用load方法可以继承，但是一般情况下不会主动去调用load方法，都是让系统自动调用
> > - load、initialize方法的区别什么？它们在category中的调用的顺序？以及出现继承时他们之间的调用过程？> 
> 答：查看下文「+load 和 +initialize」
> 
> - Category能否添加成员变量？如果可以，如何给Category添加成员变量？
> 
> 答：不能直接给Category添加成员变量，但是可以间接使用 runtime 的`objc_setAssociatedObject`实现Category有成员变量的效果。



## +load 和 +initialize

### load原理

每个类、分类的+load，在程序运行过程中加载类调用一次。跟踪 `+load` 方法调用的路径。从`_objc_init`开始：

```
- objc-os.mm	- _objc_init- load_images	- prepare_load_methods	- schedule_class_load	- add_class_to_loadable_list	- add_category_to_loadable_list- call_load_methods- call_class_loads- call_category_loads
- (*load_method)(cls, SEL_load)

// 直接自动使用 (*load_method)(cls, SEL_load) 调用  +load，而不经过消息发送机制。
```

调用顺序：

- 先调用类的+load	- 按照编译先后顺序调用（先编译，先调用）	- 调用子类的+load之前会先调用父类的+load- 再调用分类的+load	- 按照编译先后顺序调用（先编译，先调用）



### initialize 原理
initialize 会在类第一次使用时调用。当第一次向类发送消息，`objc_msgSend `的过程中，首先会寻找方法的实现地址，在`lookUpImpOrForward` 函数中:

```

IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
	// ...
    if (initialize && !cls->isInitialized()) {
        cls = initializeAndLeaveLocked(cls, inst, runtimeLock);
    }
    ...
}
```

所有未初始化过的类都会调用到`void initializeNonMetaClass(Class cls)`函数。
在这个函数中递归调用父类的`+initialize`方法：

```
void initializeNonMetaClass(Class cls)
{
    assert(!cls->isMetaClass());

    Class supercls;
    bool reallyInitialize = NO;

    supercls = cls->superclass;
    // 递归调用父类的 +initialize
    if (supercls  &&  !supercls->isInitialized()) {
        initializeNonMetaClass(supercls);
    }
    // ...    
    
    if (reallyInitialize) {
        // Exceptions: A +initialize call that throws an exception 
        // is deemed to be a complete and successful +initialize.
        //
        // Only __OBJC2__ adds these handlers. !__OBJC2__ has a
        // bootstrapping problem of this versus CF's call to
        // objc_exception_set_functions().
#if __OBJC2__
        @try
#endif
        {
            callInitialize(cls);  
        }
     }
    // ...
}
```

其中`callInitialize`调用类的 `+initialize` 方法：

```
void callInitialize(Class cls)
{
    ((void(*)(Class, SEL))objc_msgSend)(cls, SEL_initialize);
    asm("");
}
``` 

所以 `+initialize` 调用的顺序是先调用父类的`+initialize`，再调用子类的`+initialize`。

### +initialize 和 +load 区别

`+initialize` 和 `+load` 的很大区别是，`+initialize`是通过`objc_msgSend`进行调用的，所以有以下特点：
- 如果子类没有实现`+initialize`，会调用父类的`+initialize`（所以父类的`+initialize`可能会被调用多次）- 如果 Category 实现了`+initialize`，就"覆盖"类本身的`+initialize`调用。


## 如何给Category添加成员变量

```
添加关联对象void objc_setAssociatedObject(id object, const void * key,                                id value, objc_AssociationPolicy policy)获得关联对象id objc_getAssociatedObject(id object, const void * key)移除所有的关联对象void objc_removeAssociatedObjects(id object)
```

更多AssociatedObject原理参考：[捌：AssociatedObject关联对象 原理]()

