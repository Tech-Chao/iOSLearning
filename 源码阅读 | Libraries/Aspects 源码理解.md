# Aspects 源码理解

## 0.面向切面编程

> 摘自网络： 在开发过程中，面向切面编程是通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。AOP为Aspect Oriented Programming的缩写。


在 `Objective-C` 中，`runtime`保证了语言的动态性的同时，`runtime`的黑魔法之一`method swizzle`也是`Objective-C`的面向切面编程的基础，在无法看到一个类的源代码的情况下，改变方法实现或者替换方法实现的一种强大技术。

而今天的主角 `Aspects`正是基于`runtime`的`method swizzle`实现的。
## 1. 消息转发
在了解 Aspects 之前我们需要大概的了解 OC 中的消息发送机制，为学习了解 Aspects 做准备。

我们知道 OC 在发送消息时，是通过 runtime 的 `objc_msgSend(void /* id self, SEL op, ... */ )` 来发送的。在 `[receiver message]`中 objc_msgSend 会找到 receiver 的 methodlist 通过 selector 找到函数实现地址 IMP。如果 selector 有对应的 IMP ,则直接执行，如果没有，会调用`_objc_msgForward` / `_objc_msgForward_stret`函数 ,依次执行 `resolveInstanceMethod`、`forwardingTargetForSelector`、`forwardInvocation`方法。

- `resolveInstanceMethod` 动态添加一个方法实现
- `forwardingTargetForSelector` 将消息发送给其它对象处理
-  调用methodSignatureForSelector:方法，尝试获得一个方法签名。如果获取不到，则直接调用`doesNotRecognizeSelector`抛出异常。
- `forwardInvocation`最后一次机会,把前一步获取到的方法签名包装成Invocation传入，由开发者处理。
-  前面步骤如果都没有执行，则执行 `doesNotRecognizeSelector`抛出异常。


Aspects 切面编程的时候也是基于上面的消息发送机制，Aspects 在对方法进行 hook 时会进行两次 hook ，首先将需要 hook 的 `selector` 的 IMP 其指向 `_objc_msgForward` / `_objc_msgForward_stret`,且 Aspects 会 hook 当前对象的 `forwardInvocation` 的IMP，让其指向自定义的函数`__ASPECTS_ARE_BEING_CALLED__`。

 hook完之后的方法调用过程是：
 
1. 首先根据被 hook 的 `selector` 时，生成一个新的 `aliasSelector`，将类的 `forwardInvocation` 的 IMP 指向自定义的`__ASPECTS_ARE_BEING_CALLED__`方法，并将 selector 的 IMP 指向`_objc_msgForward` / `_objc_msgForward_stret`,
2. 消息转发调用到 `forwardInvocation` 发现 IMP 被指向自定义的`__ASPECTS_ARE_BEING_CALLED__`方法，执行自定义的操作（执行block）。
3. 按指定的执行时机（after、before、instead）执行 `aliasSelector`。


## 2.Aspects

[源码地址](https://github.com/steipete/Aspects)。`Aspects`是一个轻量级的框架，只有`Aspects.h`、`Aspects.m`。两个文件。但通过`Aspects`的学习让你对`runtime`的认识提升一个等级。`Aspects`通过对`NSObject`扩展各添加了一个实例、类方法，同时返回遵守`AspectToken`协议的`id`对象，用做之后的取消。

```
// 针对某个类 hook。
+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
                           withOptions:(AspectOptions)options
                            usingBlock:(id)block
                                 error:(NSError **)error;
// 针对指定的实例对象 hook。
- (id<AspectToken>)aspect_hookSelector:(SEL)selector
                           withOptions:(AspectOptions)options
                            usingBlock:(id)block
                                 error:(NSError **)error;
```

options:可选项的里面是调用切片方法的时机，之后、替换、之前、调用完自动移除。

```
typedef NS_OPTIONS(NSUInteger, AspectOptions) {
    AspectPositionAfter   = 0,            /// Called after the original implementation (default)
    AspectPositionInstead = 1,            /// Will replace the original implementation.
    AspectPositionBefore  = 2,            /// Called before the original implementation.
    
    AspectOptionAutomaticRemoval = 1 << 3 /// Will remove the hook after the first execution.
};

```
### 2.1 Aspect框架结构

在`Aspects`中所有的`hook`操作都会被封装成对象。首先我们来看下`Aspects`的类的组成。

#### 2.1.1 AspectInfo

一个`aspect`的执行环境，包含被 hook 的实例、参数和`NSInvocation`信息。遵守 AspectInfo 协议。
    
```
@interface AspectInfo : NSObject <AspectInfo>
- (id)initWithInstance:(__unsafe_unretained id)instance invocation:(NSInvocation *)invocation;
@property (nonatomic, unsafe_unretained, readonly) id instance;
@property (nonatomic, strong, readonly) NSArray *arguments;
@property (nonatomic, strong, readonly) NSInvocation *originalInvocation;
@end
```

遵守 AspectInfo 协议的对象在开发过程中作为`block`的第一个参数用来获取`Aspect`的相关信息。
```
usingBlock:^(id<AspectInfo> info) {
	NSLog(@"%@", [info instance]);
    }
```

### 2.1.2 AspectsContainer

记录一个对象或类的某个方法的所有 `aspects` 情况， 以 `aliasSelector` 为 key 使用 `objc_getAssociatedObject`绑定 AspectsContainer 实例到被 hook 的类中。

```
@interface AspectsContainer : NSObject
- (void)addAspect:(AspectIdentifier *)aspect withOptions:(AspectOptions)injectPosition;
- (BOOL)removeAspect:(id)aspect;
- (BOOL)hasAspects;
@property (atomic, copy) NSArray *beforeAspects;
@property (atomic, copy) NSArray *insteadAspects;
@property (atomic, copy) NSArray *afterAspects;
@end
```

### 2.1.3 AspectIdentifier

一个 `Aspect` 的具体内容

```
@interface AspectIdentifier : NSObject
+ (instancetype)identifierWithSelector:(SEL)selector object:(id)object options:(AspectOptions)options block:(id)block error:(NSError **)error;
- (BOOL)invokeWithInfo:(id<AspectInfo>)info;
/// 被hook的方法selector
@property (nonatomic, assign) SEL selector;
/// 执行的 block
@property (nonatomic, strong) id block;
/// block的签名,详见下文
@property (nonatomic, strong) NSMethodSignature *blockSignature;
/// 被hook的类
@property (nonatomic, weak) id object;
/// 执行的时机 options
@property (nonatomic, assign) AspectOptions options;
@end
```

### 2.1.4 AspectTracker

AspectTracker 切面的追踪，存储在全局字典 swizzledClassesDict 中，从子类向上追踪记录信息。确保类的方法只会被 hook 一次。

```
@interface AspectTracker : NSObject
- (id)initWithTrackedClass:(Class)trackedClass;
@property (nonatomic, strong) Class trackedClass;
@property (nonatomic, readonly) NSString *trackedClassName;
@property (nonatomic, strong) NSMutableSet *selectorNames;
@property (nonatomic, strong) NSMutableDictionary *selectorNamesToSubclassTrackers;
- (void)addSubclassTracker:(AspectTracker *)subclassTracker hookingSelectorName:(NSString *)selectorName;
- (void)removeSubclassTracker:(AspectTracker *)subclassTracker hookingSelectorName:(NSString *)selectorName;
- (BOOL)subclassHasHookedSelectorName:(NSString *)selectorName;
- (NSSet *)subclassTrackersHookingSelectorName:(NSString *)selectorName;
@end
```
## 2.2 Hook过程

在 `Aspects` 中提供了一个针对对象、和类 hook 的方法,其核心代码都是通过 `aspect_add`来进行 hook ：

核心代码：

```
static id aspect_add(id self, SEL selector, AspectOptions options, id block, NSError **error) {
    __block AspectIdentifier *identifier = nil;
    aspect_performLocked(^{
        if (aspect_isSelectorAllowedAndTrack(self, selector, options, error)) {
            AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);
            identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];
            if (identifier) {
                [aspectContainer addAspect:identifier withOptions:options];

                // Modify the class to allow message interception.
                aspect_prepareClassAndHookSelector(self, selector, error);
            }
        }
    });
    return identifier;
}
```

代码简化了一下去除了一些断言等判断，Aspect 通过 `aspect_performLocked `的 block 使用自旋锁/OSSpinLock 保证这段代码的线程安全（**注1**）。

首先调用 `aspect_isSelectorAllowedAndTrack` 方法过滤下无效的hook 方法：

- 去除 hook 方法的黑名单（retain, release， autorelease, forwardInvocation等方法），

```
    //  判断 hook 方法是否在黑名单上 [@"retain", @"release", @"autorelease", @"forwardInvocation:"]
    NSString *selectorName = NSStringFromSelector(selector);
    if ([disallowedSelectorList containsObject:selectorName]) {
        return NO;
    }
```

- 限制hook dealloc方法时判断只能在之前插入。

```
  // 检测 hook 的 dealloc 必须逻辑之前执行。
    AspectOptions position = options&AspectPositionFilter;
    if ([selectorName isEqualToString:@"dealloc"] && position != AspectPositionBefore) {
        return NO;
    }
```
- 检测被 hook 的方法在 self 和 self.class 是否存在。

```
    if (![self respondsToSelector:selector] && ![self.class instancesRespondToSelector:selector]) {
        return NO;
    }
```

- 针对类对象的层级结构(父类、子类)进行判断，防止同一方法被hook多次。

```
    // 判断是否是元类
      if (class_isMetaClass(object_getClass(self))) {
        Class klass = [self class];
		 //	 生成一个全局的类记录字典
        NSMutableDictionary *swizzledClassesDict = aspect_getSwizzledClassesDict();

        Class currentClass = [self class];
		// 判断方法在子类中是否已经被 hook 了。保证一个方法在类的层级里面只能被 hook 一次
        AspectTracker *tracker = swizzledClassesDict[currentClass];
        if ([tracker subclassHasHookedSelectorName:selectorName]) {
            NSSet *subclassTracker = [tracker subclassTrackersHookingSelectorName:selectorName];
            NSSet *subclassNames = [subclassTracker valueForKey:@"trackedClassName"];
            NSString *errorDescription = [NSString stringWithFormat:@"Error: %@ already hooked subclasses: %@. A method can only be hooked once per class hierarchy.", selectorName, subclassNames];
            AspectError(AspectErrorSelectorAlreadyHookedInClassHierarchy, errorDescription);
            return NO;
        }

// 沿着 superclass 指针一直向上到 NSObject 根类，保证一个方法在一个类的层级里面只能被 hook 一次
        do {
            tracker = swizzledClassesDict[currentClass];
            if ([tracker.selectorNames containsObject:selectorName]) {
                if (klass == currentClass) {
                    // Already modified and topmost!
                    return YES;
                }
                NSString *errorDescription = [NSString stringWithFormat:@"Error: %@ already hooked in %@. A method can only be hooked once per class hierarchy.", selectorName, NSStringFromClass(currentClass)];
                AspectError(AspectErrorSelectorAlreadyHookedInClassHierarchy, errorDescription);
                return NO;
            }
        } while ((currentClass = class_getSuperclass(currentClass)));

		// 沿着继承链，使用 AspectTracker 记录被 hook 的信息，
        currentClass = klass;
        AspectTracker *subclassTracker = nil;
        do {
            tracker = swizzledClassesDict[currentClass];
            if (!tracker) {
                tracker = [[AspectTracker alloc] initWithTrackedClass:currentClass];
                swizzledClassesDict[(id<NSCopying>)currentClass] = tracker;
            }
            if (subclassTracker) {
                [tracker addSubclassTracker:subclassTracker hookingSelectorName:selectorName];
            } else {
                [tracker.selectorNames addObject:selectorName];
            }

            // All superclasses get marked as having a subclass that is modified.
            subclassTracker = tracker;
        }while ((currentClass = class_getSuperclass(currentClass)));
	} 
```

**注2：object_getClass(self) 和 self.class 区别。**

```
static NSString *const AspectsMessagePrefix = @"aspects_";
```
满足上诉条件后,使用 AspectsMessagePrefix 拼接上 selector 生成的 aliasSelector 为key，  `objc_getAssociatedObject` 获取或创建 AspectsContainer 对象来存放被 hook 方法的所有 `aspect`。


```
+ (instancetype)identifierWithSelector:(SEL)selector
                                object:(id)object
                               options:(AspectOptions)options
                                 block:(id)block
                                 error:(NSError **)error;
```

创建 AspectIdentifier 对象时，通过 `aspect_blockMethodSignature`获取 block 的签名同 selector 的签名调用`aspect_isCompatibleBlockSignature`进行比对，确保参数一致，创建好的 AspectIdentifier 对象，存放在 AspectsContainer 中，然后调用 `aspect_prepareClassAndHookSelector`开始 hook 方法。


```
// Block internals.
typedef NS_OPTIONS(int, AspectBlockFlags) {
	AspectBlockFlagsHasCopyDisposeHelpers = (1 << 25),
	AspectBlockFlagsHasSignature          = (1 << 30)
};

typedef struct _AspectBlock {
	__unused Class isa;
	AspectBlockFlags flags;
	__unused int reserved;
	void (__unused *invoke)(struct _AspectBlock *block, ...);
	struct {
		unsigned long int reserved;
		unsigned long int size;
		// requires AspectBlockFlagsHasCopyDisposeHelpers
		void (*copy)(void *dst, const void *src);
		void (*dispose)(const void *);
		// requires AspectBlockFlagsHasSignature
		const char *signature;
		const char *layout;
	} *descriptor;
	// imported variables
} *AspectBlockRef;


static NSMethodSignature *aspect_blockMethodSignature(id block, NSError **error) {
    AspectBlockRef layout = (__bridge void *)block;
	if (!(layout->flags & AspectBlockFlagsHasSignature)) {
        NSString *description = [NSString stringWithFormat:@"The block %@ doesn't contain a type signature.", block];
        AspectError(AspectErrorMissingBlockSignature, description);
        return nil;
    }
	void *desc = layout->descriptor;
	desc += 2 * sizeof(unsigned long int);
	if (layout->flags & AspectBlockFlagsHasCopyDisposeHelpers) {
		desc += 2 * sizeof(void *);
    }
	if (!desc) {
        NSString *description = [NSString stringWithFormat:@"The block %@ doesn't has a type signature.", block];
        AspectError(AspectErrorMissingBlockSignature, description);
        return nil;
    }
	const char *signature = (*(const char **)desc);
	return [NSMethodSignature signatureWithObjCTypes:signature];
}

```
AspectBlockRef 同 block 的结构一致，所以作者在 ` aspect_blockMethodSignature` 中 将block 强转为自定义的结构体 AspectBlockRef ，根据 flags 判断是否存在签名和block是否捕获外部变量， 进行偏移得出 block 签名。

```

static BOOL aspect_isCompatibleBlockSignature(NSMethodSignature *blockSignature, id object, SEL selector, NSError **error) {
    NSCParameterAssert(blockSignature);
    NSCParameterAssert(object);
    NSCParameterAssert(selector);

    BOOL signaturesMatch = YES;
    NSMethodSignature *methodSignature = [[object class] instanceMethodSignatureForSelector:selector];
    if (blockSignature.numberOfArguments > methodSignature.numberOfArguments) {
        signaturesMatch = NO;
    }else {
        if (blockSignature.numberOfArguments > 1) {
            const char *blockType = [blockSignature getArgumentTypeAtIndex:1];
            if (blockType[0] != '@') {
                signaturesMatch = NO;
            }
        }
        // Argument 0 is self/block, argument 1 is SEL or id<AspectInfo>. We start comparing at argument 2.
        // The block can have less arguments than the method, that's ok.
        if (signaturesMatch) {
            for (NSUInteger idx = 2; idx < blockSignature.numberOfArguments; idx++) {
                const char *methodType = [methodSignature getArgumentTypeAtIndex:idx];
                const char *blockType = [blockSignature getArgumentTypeAtIndex:idx];
                // Only compare parameter, not the optional type data.
                if (!methodType || !blockType || methodType[0] != blockType[0]) {
                    signaturesMatch = NO; break;
                }
            }
        }
    }

    if (!signaturesMatch) {
        NSString *description = [NSString stringWithFormat:@"Block signature %@ doesn't match %@.", blockSignature, methodSignature];
        AspectError(AspectErrorIncompatibleBlockSignature, description);
        return NO;
    }
    return YES;
}
```
根据 selector 获取方法签名，对比 block 的方法签名的参数个数和参数类型。返回方法签名v匹配，注意在 OC 方法中 隐含 2 个默认参数 self 、 _cmd，所以真正比对参数是从 selector 的第二个参数开始。


在 aspect_prepareClassAndHookSelector 会有两次 hook， 一次将 class 的 forwardInvocation 指向 `__ASPECTS_ARE_BEING_CALLED__`,另一次将 selector 指向 `_objc_msgForward/_objc_msgForward_stret` 进行消息转发。

```
static Class aspect_hookClass(NSObject *self, NSError **error) {
    NSCParameterAssert(self);
	Class statedClass = self.class;
	Class baseClass = object_getClass(self);
	NSString *className = NSStringFromClass(baseClass);

    // 判断该类是否已经被hook过了
	if ([className hasSuffix:AspectsSubclassSuffix]) {
		return baseClass;

	// 判断是否是元类
	}else if (class_isMetaClass(baseClass)) {
        return aspect_swizzleClassInPlace((Class)self);
	// 兼容性判断是否是 KVO 过的子类
    }else if (statedClass != baseClass) {
        return aspect_swizzleClassInPlace(baseClass);
    }
	

    // 动态的创建子类
	const char *subclassName = [className stringByAppendingString:AspectsSubclassSuffix].UTF8String;
	Class subclass = objc_getClass(subclassName);

	if (subclass == nil) {
		subclass = objc_allocateClassPair(baseClass, subclassName, 0);
		if (subclass == nil) {
            NSString *errrorDesc = [NSString stringWithFormat:@"objc_allocateClassPair failed to allocate class %s.", subclassName];
            AspectError(AspectErrorFailedToAllocateClassPair, errrorDesc);
            return nil;
        }
		// hook 子类的 forwardInvocation 方法 指向 __ASPECTS_ARE_BEING_CALLED__
		aspect_swizzleForwardInvocation(subclass);
		// 隐藏子类化的类，当外部调用 class 返回 statedClass类型。
		aspect_hookedGetClass(subclass, statedClass);
		aspect_hookedGetClass(object_getClass(subclass), statedClass);
		// 注册子类
		objc_registerClassPair(subclass);
	}
	// 将 self 的 isa 设置为子类，
	object_setClass(self, subclass);
	return subclass;
}
```

第一次 hook ，Aspects 会类似 KVO 的实现，会动态创建一个子类,将当前对象变成一个 subclass 的实例，同时对于外部使用者而言，又能把它继续当成原对象在使用，而且所有的 hook 操作都发生在子类，这样做的好处是你不需要去更改对象本身的。在`aspect_hookClass` 函数中如果 hook 的是 类是元类或者是KVO时产生的子类则将类本身的 forwardInvocation 实现指向我们自定义的 `__ASPECTS_ARE_BEING_CALLED__`函数，如果是普通的类则创建子类，然后 hook 子类的 forwardInvocation 将实现指向我们自定义的`__ASPECTS_ARE_BEING_CALLED__`函数。在 `__ASPECTS_ARE_BEING_CALLED__` 中统一处理被 hook 的方法。

```
static void aspect_prepareClassAndHookSelector(NSObject *self, SEL selector, NSError **error) {
    NSCParameterAssert(selector);
    Class klass = aspect_hookClass(self, error);
    Method targetMethod = class_getInstanceMethod(klass, selector);
    IMP targetMethodIMP = method_getImplementation(targetMethod);
    if (!aspect_isMsgForwardIMP(targetMethodIMP)) {
        // Make a method alias for the existing method implementation, it not already copied.
        const char *typeEncoding = method_getTypeEncoding(targetMethod);
        SEL aliasSelector = aspect_aliasForSelector(selector);
        if (![klass instancesRespondToSelector:aliasSelector]) {
            __unused BOOL addedAlias = class_addMethod(klass, aliasSelector, method_getImplementation(targetMethod), typeEncoding);
            NSCAssert(addedAlias, @"Original implementation for %@ is already copied to %@ on %@", NSStringFromSelector(selector), NSStringFromSelector(aliasSelector), klass);
        }

        // We use forwardInvocation to hook in.
        class_replaceMethod(klass, selector, aspect_getMsgForwardIMP(self, selector), typeEncoding);
        AspectLog(@"Aspects: Installed hook for -[%@ %@].", klass, NSStringFromSelector(selector));
}
```
第二次 hook 将 selector 指向 `_objc_msgForward/_objc_msgForward_stret`，同时将原先的 selector copy一份 aliasSelector 添加到类中。

`_objc_msgForward` 与 `_objc_msgForward_stret` 使用哪一个和 CPU 的状态寄存器的内容相关，具体可以查看下[JSPatch-实现原理详解](https://github.com/bang590/JSPatch/wiki/JSPatch-%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E8%AF%A6%E8%A7%A3#1special-struct)

至此，AspectIdentifier 对象生成完毕且存放到了 AspectsContainer 容器中。

接下来，我们来看看当被 hook 的方法被调用时发生了什么。根据之前的介绍。我们知道一系列 swizzle 之后，方法会 `__ASPECTS_ARE_BEING_CALLED__` 函数中执行。

```
// This is the swizzled forwardInvocation: method.
static void __ASPECTS_ARE_BEING_CALLED__(__unsafe_unretained NSObject *self, SEL selector, NSInvocation *invocation) {
    NSCParameterAssert(self);
    NSCParameterAssert(invocation);

    SEL originalSelector = invocation.selector;
	SEL aliasSelector = aspect_aliasForSelector(invocation.selector);
    invocation.selector = aliasSelector;
	// 获取对象、类上面的所有 aspect
    AspectsContainer *objectContainer = objc_getAssociatedObject(self, aliasSelector);
    AspectsContainer *classContainer = aspect_getContainerForClass(object_getClass(self), aliasSelector);
    AspectInfo *info = [[AspectInfo alloc] initWithInstance:self invocation:invocation];
    NSArray *aspectsToRemove = nil;

    // Before hooks.
    aspect_invoke(classContainer.beforeAspects, info);
    aspect_invoke(objectContainer.beforeAspects, info);

    // Instead hooks.
    BOOL respondsToAlias = YES;
    if (objectContainer.insteadAspects.count || classContainer.insteadAspects.count) {
        aspect_invoke(classContainer.insteadAspects, info);
        aspect_invoke(objectContainer.insteadAspects, info);
    }else {
        Class klass = object_getClass(invocation.target);
        do {
            if ((respondsToAlias = [klass instancesRespondToSelector:aliasSelector])) {
                [invocation invoke];
                break;
            }
        }while (!respondsToAlias && (klass = class_getSuperclass(klass)));
    }

    // After hooks.
    aspect_invoke(classContainer.afterAspects, info);
    aspect_invoke(objectContainer.afterAspects, info);

    // If no hooks are installed, call original implementation (usually to throw an exception)
    if (!respondsToAlias) {
        invocation.selector = originalSelector;
        SEL originalForwardInvocationSEL = NSSelectorFromString(AspectsForwardInvocationSelectorName);
        if ([self respondsToSelector:originalForwardInvocationSEL]) {
            ((void( *)(id, SEL, NSInvocation *))objc_msgSend)(self, originalForwardInvocationSEL, invocation);
        }else {
            [self doesNotRecognizeSelector:invocation.selector];
        }
    }

    // Remove any hooks that are queued for deregistration.
    [aspectsToRemove makeObjectsPerformSelector:@selector(remove)];
}
```

首先根据 selector 生成的 aliasSelector 获取所有的 AspectsContainer， 首先执行 AspectsContainer 所有
的 beforeAspects， 判断是否存在 insteadAspects ，存在则执行 insteadAspects，不存在执行原有方法， 再执行 afterAspects 。最后根据记录的 aspectsToRemove 移除执行完的 aspect。

宏定义执行 aspect：

```
#define aspect_invoke(aspects, info) \
for (AspectIdentifier *aspect in aspects) {\
    [aspect invokeWithInfo:info];\
    if (aspect.options & AspectOptionAutomaticRemoval) { \
        aspectsToRemove = [aspectsToRemove?:@[] arrayByAddingObject:aspect]; \
    } \
}
```

作者在这是使用宏定义作用主要是为了获取一个清晰的调用堆栈。宏定义中遍历 aspects 调用 AspectIdentifier 的 invokeWithInfo 方法。

```
- (BOOL)invokeWithInfo:(id<AspectInfo>)info {
    NSInvocation *blockInvocation = [NSInvocation invocationWithMethodSignature:self.blockSignature];
    NSInvocation *originalInvocation = info.originalInvocation;
    NSUInteger numberOfArguments = self.blockSignature.numberOfArguments;

    // 之前已经做过判断，作者在加判断是因为强迫症
    if (numberOfArguments > originalInvocation.methodSignature.numberOfArguments) {
        AspectLogError(@"Block has too many arguments. Not calling %@", info);
        return NO;
    }

	// block 插入第一个参数为  AspectInfo
    if (numberOfArguments > 1) {
        [blockInvocation setArgument:&info atIndex:1];
    }
    
	// 将 originalInvocation 的参数 传入 blockInvocation 中。
	void *argBuf = NULL;
    for (NSUInteger idx = 2; idx < numberOfArguments; idx++) {
        const char *type = [originalInvocation.methodSignature getArgumentTypeAtIndex:idx];
		NSUInteger argSize;
		NSGetSizeAndAlignment(type, &argSize, NULL);
        
		if (!(argBuf = reallocf(argBuf, argSize))) {
            AspectLogError(@"Failed to allocate memory for block invocation.");
			return NO;
		}
		[originalInvocation getArgument:argBuf atIndex:idx];
		[blockInvocation setArgument:argBuf atIndex:idx];
    }
    
    [blockInvocation invokeWithTarget:self.block];
    
    if (argBuf != NULL) {
        free(argBuf);
    }
    return YES;
}
```

#注
- 注1：`Aspects`使用`OSSpinLock`来确保执行时的线程安全，虽然`OSSpinLock`已被证实不在安全。参考[不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)，但是维护人员认为`OSSpinLock`目前还可以挣扎下，决定在`iOS9`过期后再替换`os_unfair_lock`。

- 注2：object_getClass(self) 和 self.class 区别。

```

当对象调用 object_getClass(self)、self.class 结果一致,返回当前对象的类。
当类调用 object_getClass(self)、self.class 分别返回 类的元类、自身。源码如下

// 返回 isa 的指针指向的类
Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();
    else return Nil;
}

// 类调用时返回自身
+ (Class)class {
    return self;
}
// 对象调用时调用object_getClass
- (Class)class {
    return object_getClass(self);
}

```
