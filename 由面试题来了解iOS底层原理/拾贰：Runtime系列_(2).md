# 拾贰：Runtime系列_(2)

## 方法的调用过程

在 OC 中无论对象方法还是类方法的调用最终都离不开`objc_msgsend()`函数，先看一段简单的代码：

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/objc_msgsend.png?raw=true)

Tips: 在调用`objc_msgsend()`函数的时候，Xcode不会给我们提示参数，需要自己修改 Xcode ，在项目中的build setting ->搜索objc_msgSend -> 把YES 改为 NO。

可以看到方法都会转换成`objc_msgsend()`函数，那么`objc_msgsend()`调用的流程又是怎么样的呢？


在 obj4 的 runtime 源码中我们可以在`objc-msg-arm.s`找到`objc_msgsend() `函数的入口:

```
/********************************************************************
 *
 * id objc_msgSend(id self, SEL _cmd, ...);
 * IMP objc_msgLookup(id self, SEL _cmd, ...);
 * 
 * objc_msgLookup ABI:
 * IMP returned in r12
 * Forwarding returned in Z flag
 * r9 reserved for our use but not used
 *
 ********************************************************************/

	ENTRY _objc_msgSend
	ENTRY _objc_msgSend
	
	cbz	r0, LNilReceiver_f // CBZ 比较（Compare），如果 r0 结果为零（Zero）就跳转 LNilReceiver_f

	ldr	r9, [r0]		// r9 = self->isa  // 将 r0 的 isa 赋值r9
	GetClassFromIsa			// r9 = class// 通过 isa 计算出 类的地址
	CacheLookup NORMAL	// 开始方法缓存查找
	// cache hit, IMP in r12, eq already set for nonstret forwarding
	bx	r12			// call imp 命中的话直接使用 imp 调用方法实现

	CacheLookup2 NORMAL 
	// cache miss
	ldr	r9, [r0]		// r9 = self->isa
	GetClassFromIsa			// r9 = class
	b	__objc_msgSend_uncached 

LNilReceiver:
	// r0 is already zero
	mov	r1, #0
	mov	r2, #0
	mov	r3, #0
	FP_RETURN_ZERO
	bx	lr	

	END_ENTRY _objc_msgSend
```

一堆的汇编代码,个人表示相当无力结合网上资料的大概的理解是显示缓存查找如果未命中就会走到`__objc_msgSend_uncached `调用 `MethodTableLookup` 在通过 `__class_lookupMethodAndLoadCache3`函数寻找方法的实现。

```
IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
{
    return lookUpImpOrForward(cls, sel, obj, 
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}


IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    IMP imp = nil;
    bool triedResolver = NO;

    runtimeLock.assertUnlocked();

    // 优化再次缓存查找
    if (cache) {
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }


    runtimeLock.lock();
    checkIsKnownClass(cls);

    if (!cls->isRealized()) {
        cls = realizeClassMaybeSwiftAndLeaveLocked(cls, runtimeLock);
        // runtimeLock may have been dropped but is now locked again
    }

    if (initialize && !cls->isInitialized()) {
        cls = initializeAndLeaveLocked(cls, inst, runtimeLock);
    }

 retry:    
    runtimeLock.assertLocked();
	// 首先查找类的 cache
    imp = cache_getImp(cls, sel);
    if (imp) goto done;

	// cache 未命中查找类的方法列表 class_rw_t 中的 methods
    {
        Method meth = getMethodNoSuper_nolock(cls, sel);
        if (meth) {
            log_and_fill_cache(cls, meth->imp, sel, inst, cls);
            imp = meth->imp;
            goto done;
        }
    }
 	// 类本身为找到方法，向上 父类的 cache 和 methods 中寻找
    {
        unsigned attempts = unreasonableClassCount();
        for (Class curClass = cls->superclass;
             curClass != nil;
             curClass = curClass->superclass)
        {
            // Halt if there is a cycle in the superclass chain.
            if (--attempts == 0) {
                _objc_fatal("Memory corruption in class list.");
            }
            
            // Superclass cache.
            imp = cache_getImp(curClass, sel);
            if (imp) {
                if (imp != (IMP)_objc_msgForward_impcache) {
                    // Found the method in a superclass. Cache it in this class.
                    log_and_fill_cache(cls, imp, sel, inst, curClass);
                    goto done;
                }
                else {
                    // Found a forward:: entry in a superclass.
                    // Stop searching, but don't cache yet; call method 
                    // resolver for this class first.
                    break;
                }
            }
            
            // Superclass method list.
            Method meth = getMethodNoSuper_nolock(curClass, sel);
            if (meth) {
                log_and_fill_cache(cls, meth->imp, sel, inst, curClass);
                imp = meth->imp;
                goto done;
            }
        }
    }

	// 方法还未找到进行一次动态方法解析
    if (resolver  &&  !triedResolver) {
        runtimeLock.unlock();
        resolveMethod(cls, sel, inst);
        runtimeLock.lock();
        triedResolver = YES;
        goto retry;
    }

	// 动态方法解析无效的话最后进行消息转发
    imp = (IMP)_objc_msgForward_impcache;
    cache_fill(cls, sel, imp, inst);

 done:
    runtimeLock.unlock();

    return imp;
}


```

总结下消息发送（以对象方法为例，类方法从元类中查找）的流程：

1. 首先判断消息接受者 receiver 是否为空，如果为nil直接退出，否则进入第2步
2. 查找 receiver 的类中的方法缓存 cache,缓存命中，直接调用方法，未命中进入第3步
3. 查找 receiver 中 `class_rw_t` 中的 `methods`,查找命中则调用方法并缓存方法到 receiver 类的 cache 中，未命中进入第4步
4. 通过 receiver 类中的 superclass 指针继续向上寻找，首先寻找`superclass`中 cache，缓存命中调用方法，并将方法缓存到 receiver 类的 cache 中，否则进入第5步。
5. 查找 receiver 父类 `class_rw_t` 中的 `methods`，查找命中调用方法，并将方法缓存到 receiver 的 superclass 的 cache 和 receiver 类的 cache 中。否则进入第6步
6. 按照 receiver 父类 的 superclass 指针一直向上按照 cache、mehtods的寻找顺序寻找直到找到方法。否则进入第7步骤
7. 直到根类都未找到方法，系统开始进行动态方法解析。
8. 动态方法解析无效开始进行消息转发。

以上，就是方法调用的整个大概过程。

## 动态方法解析

那么在方法未找到时，进行动态方法解析`resolveMethod`：

```
static void resolveMethod(Class cls, SEL sel, id inst)
{
    runtimeLock.assertUnlocked();
    assert(cls->isRealized());
	//  判断是否元类，调用resolveInstanceMethod、resolveClassMethod方法
    if (! cls->isMetaClass()) {
        resolveInstanceMethod(cls, sel, inst);
    } 
    else {
        resolveClassMethod(cls, sel, inst);
        if (!lookUpImpOrNil(cls, sel, inst, 
                            NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
        {
            resolveInstanceMethod(cls, sel, inst);
        }
    }
}

static void resolveInstanceMethod(Class cls, SEL sel, id inst)
{
    runtimeLock.assertUnlocked();
    assert(cls->isRealized());

    if (! lookUpImpOrNil(cls->ISA(), SEL_resolveInstanceMethod, cls, 
                         NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
    {
        // Resolver not implemented.
        return;
    }

    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
	// 相当于调用类中的 +resolveInstanceMethod 方法
    bool resolved = msg(cls, SEL_resolveInstanceMethod, sel);

	/// ..
}

```

所以动态方法解析需要我们在类中重写`+ (BOOL)resolveInstanceMethod:(SEL)sel`（对象方法的动态解析）、`+ (BOOL)resolveClassMethod:(SEL)sel`（类方法的动态解析）。我们以对象方法的动态解析为例定义 Person 类声明了`run`，但没有具体实现,重写`resolveClassMethod `代码结构大概如下：

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/resolveInstanceMethod.png?raw=true)

```
    Person *p = [[Person alloc] init];
    [p run];
    // 输出了 other method called
```

可以看到我们通过`class_addMethod(Class cls, SEL name, IMP imp, const char *types) `动态的为 Person 类添加了方法的实现。

## 消息转发

如果动态方法解析未做处理没有重写相关的方法，那么会进入的消息转发阶段。会进行消息转发阶段，调用`_objc_msgForward_impcache`,查看汇编的调用过程`__objc_msgForward_impcache` ->  `__objc_msgForward` -> `__objc_forward_handler`，但在源码中只找到了一段默认的实现:

```
__attribute__((noreturn)) void 
objc_defaultForwardHandler(id self, SEL sel)
{
    _objc_fatal("%c[%s %s]: unrecognized selector sent to instance %p "
                "(no message forward handler is installed)", 
                class_isMetaClass(object_getClass(self)) ? '+' : '-', 
                object_getClassName(self), sel_getName(sel), self);
}
void *_objc_forward_handler = (void*)objc_defaultForwardHandler;

// 在平时开发中最常见到的方法未找到错误的log就是在这里输出的
```

接下来通过一个对象方法的消息转发例子来看看消息转发的工作过程：

- `Person`类没有实现 `-test`,我们通过重写`forwardingTargetForSelector`返回 `Animal` 的对象来处理`-test`方法:

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/forward_person.png?raw=true)

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/forward_animal.png?raw=true)

```
	Person *p = [Person new];
	[p test];     
	// 输出： Animal test
```

- `Person` 没有重写`forwardingTargetForSelector`或者返回`nil`，重写了`methodSignatureForSelector`、`forwardInvocation:(NSInvocation *)anInvocation`:

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/forward_signature.png?raw=true)

效果和上面直接重写`forwardingTargetForSelector`返回 Animal 的对象一样。整个消息转发的流程：

![](https://raw.githubusercontent.com/PhoenixiOSer/iOSLearning/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/forward_flow.png?raw=true)
 

### NSInvocation

NSInvocation对象包装了方法名、参数、返回值等，也就是说，在forwardInvocation函数中我们可以对方法进行最后的修改。我们可以在`forwardInvocation `为所欲为，任意的修改 target、方法、参数等。


**tips: 类方法也存在动态方法解析及消息转发，主要的变动在于上述的`-`变换成`+`，从元类中开始消息发送、动态方法解析、消息转发的机制。**
