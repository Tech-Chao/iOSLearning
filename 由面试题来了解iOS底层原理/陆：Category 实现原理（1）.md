# 陆：Category 实现原理（1）

>- Category的实现原理? 
> 
> 答：Category编译之后的底层结构是struct category_t，里面存储着分类的对象方法、类方法、属性、协议信息。在程序运行的时候，runtime会将Category的数据，合并到类信息中（类对象、元类对象中）详细的请看下方 『Category 方法的存放』
> 
> - Category和Class Extension的区别是什么？
> 
> 答：Class Extension在编译的时候，它的数据就已经包含在类信息中Category是在运行时，才会将数据合并到类信息中
> 



在项目中 Category 一般用来划分组合业务相关的代码，在不知道某个类的具体实现时添加方法。下面是示例代码：

```
// Person类
@interface Person : NSObject
- (void)run;
@end
@implementation Person
- (void)run {
    NSLog(@"%s",__func__);
}
@end

// Person Eat分类
@interface Person (Eat)
- (void)eat;
@end
@implementation Person (Eat)
- (void)eat {
    NSLog(@"%s",__func__);
}
@end

// Person Test分类
@interface Person (Test)<NSCopying,NSCoding>

@property (nonatomic, assign) int age;
- (void)test;
+ (void)sleep;

@end

@implementation Person (Test)
- (void)test {
    NSLog(@"%s",__func__);
    
}
+ (void)sleep {
    NSLog(@"%s",__func__);
}
@end



Person * p= [[Person alloc]init];

[Person sleep];
[p run];

[p eat];
[Person test];
```
[贰：对象的isa指针指向哪里？](https://github.com/PhoenixiOSer/iOSLearningManual/blob/master/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/%E8%B4%B0%EF%BC%9A%E5%AF%B9%E8%B1%A1%E7%9A%84isa%E6%8C%87%E9%92%88%E6%8C%87%E5%90%91%E5%93%AA%E9%87%8C%EF%BC%9F.md)、[叁：class 和 meta-class的结构](https://github.com/PhoenixiOSer/iOSLearningManual/blob/master/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/%E5%8F%81%EF%BC%9Aclass%20%E5%92%8C%20meta-class%E7%9A%84%E7%BB%93%E6%9E%84.md)我们知道类中存放着对象方法，元类中存放了类方法。实例、类、元类则以 isa 指针指向来相互关联。

而在 OC 调用方法相当于使用 `objc_msgSend` 向 receiver(接受者) 发送消息。如 `[p run];`是向 p 发送了 run 的消息，即`objc_msgSend(p,@selector(run))`，如果是类方法则是`objc_msgSend(Person,@selector(run))`。

我们可以通过 p 的 isa 指向的 Person 类中寻找的对象方法。Person 类对象的 isa 指向的元类中寻找类方法。 那么 Category 中给添加的实例方法、类方法又存放在哪？

## Category 方法的存放

其实 Category 定义的方法其实和在类中定义方法一样都是存放在类对象 `class_rw_t` 的methods中（实例方法）、元类对象 `class_rw_t` 的methods（类方法）中。 

Category 的方法时如何以及何时加载到类中的呢？对此首先我们需要了解 Category 的在底层的数据结构是什么样的。 我们对 `Person+Test.m` 使用 clang 编译生成 cpp 文件。

```
// cd 到文件所在目录，Terminal中输入
 xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc Person+Test.m
```

在生成的 Person+Test.cpp 文件中可以看到 Category 的结构：

```
struct _category_t {
	const char *name;
	struct _class_t *cls;
	const struct _method_list_t *instance_methods;
	const struct _method_list_t *class_methods;
	const struct _protocol_list_t *protocols;
	const struct _prop_list_t *properties;
};
```

我们可以看到 Category 在 C/C++ 的也是一个结构体，`_category_t `包含了 类名、主类对象、实例方法、类方法、协议、属性。而查看 obj4 源码中`struct category_t`结构和编译后的代码也是相似的;
其中`instanceProperties、_classProperties` 相当于是properties。

![](https://github.com/PhoenixiOSer/iOSLearningManual/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/Cateory_obj4.png?raw=true)

接下来看看对 Person+Test 结构体的赋值：

```
static struct _category_t _OBJC_$_CATEGORY_Person_$_Test __attribute__ ((used, section ("__DATA,__objc_const"))) = 
{
	"Person",
	0, // &OBJC_CLASS_$_Person,
	(const struct _method_list_t *)&_OBJC_$_CATEGORY_INSTANCE_METHODS_Person_$_Test,
	(const struct _method_list_t *)&_OBJC_$_CATEGORY_CLASS_METHODS_Person_$_Test,
	(const struct _protocol_list_t *)&_OBJC_CATEGORY_PROTOCOLS_$_Person_$_Test,
	(const struct _prop_list_t *)&_OBJC_$_PROP_LIST_Person_$_Test,
};

/**
 _OBJC_$_CATEGORY_Person_$_Test 对应的复制
name 				: "Person"
cls 				: 0 //会指向 OBJC_CLASS_$_Person
instance_methods	: _OBJC_$_CATEGORY_INSTANCE_METHODS_Person_$_Test
class_methods		: _OBJC_$_CATEGORY_CLASS_METHODS_Person_$_Test
protocols			: _OBJC_CATEGORY_PROTOCOLS_$_Person_$_Test
properties			: _OBJC_$_PROP_LIST_Person_$_Test


_OBJC_$_CATEGORY_INSTANCE_METHODS_Person_$_Test
_OBJC_$_CATEGORY_CLASS_METHODS_Person_$_Test
_OBJC_CATEGORY_PROTOCOLS_$_Person_$_Test
_OBJC_$_PROP_LIST_Person_$_Test
在源码中都有定义，这里就不展开细看了，有兴趣的同学可以自己编译看下。
*/
```

以上就是 Category 在 C/C++ 的结构。那么 Category 中的方法时在何时又是如何添加到主类（元类）中的呢？

在 objc4 的源码中，跟踪 objc-os.mm、 objc-runtime-new.mm文件中的相关方法。我们可以一窥究竟：

- objc-os.mm
	1. `_objc_init`	2. `map_images`	3. `map_images_nolock`- objc-runtime-new.mm	1. `_read_images`
	2. `remethodizeClass`	3. `attachCategories`	4. `attachLists`	5. `realloc、memmove、 memcpy`

其中 Category 中的方法会在 `remethodizeClass` 开始对类重新整合方法。
remethodizeClass 源码：

```
static void remethodizeClass(Class cls)
{
    category_list *cats;
    bool isMeta;

    runtimeLock.assertLocked();
	// 判断 remethodize 的类是否是元类
    isMeta = cls->isMetaClass();

    // Re-methodizing: check for more categories
    if ((cats = unattachedCategoriesForClass(cls, false/*not realizing*/))) {
        if (PrintConnecting) {
            _objc_inform("CLASS: attaching categories to class '%s' %s", 
                         cls->nameForLogging(), isMeta ? "(meta)" : "");
        }
        // 将 cls 的所有分类 attach 到主类中
        attachCategories(cls, cats, true /*flush caches*/);        
        free(cats);
    }
}
```

attachCategories正式将 Category 结构体中的元素（方法、协议、属性等）复制到主类中， attachCategories 函数源码：

```
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    bool isMeta = cls->isMetaClass();

	// 方法列表（二维数组,每个一维数组代表某一个分类的所有方法）
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    // 属性列表（二维数组,每个一维数组代表某一个分类的所有属性）
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    // 协议列表（二维数组,每个一维数组代表某一个分类的所有协议）
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    // 遍历所有的分类数组， 注意： i--，顺序是从后往前遍历。
    while (i--) {
    	 // 获取某个分类
        auto& entry = cats->list[i];
		 // 获取分类中的所有对象方法 or 类方法
        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }
		// 获取某个分类中的所有属性
        property_list_t *proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists[propcount++] = proplist;
        }
		// 获取某个分类中的所有协议
        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }
	
    auto rw = cls->data();

	
    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    // 将方法、属性、协议 添加到类的 `class_rw` 中
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);

    rw->properties.attachLists(proplists, propcount);
    free(proplists);

    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}
```



`class_rw` 的 `methods.attachLists` 进行真正的分类方法添加：

```
// addedLists （可能是对象方法、也可能是类方法，下面以对象方法注释）
void attachLists(List* const * addedLists, uint32_t addedCount) {
        if (addedCount == 0) return;

        if (hasArray()) {
            // many lists -> many lists
            // 原先主类中定义的对象方法数量
            uint32_t oldCount = array()->count;
            // 所有分类新增的对象方法
            uint32_t newCount = oldCount + addedCount;
            // 对 array 进行扩容
            setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
            array()->count = newCount;
            // 将原先类中的方法后移
            memmove(array()->lists + addedCount, array()->lists, 
                    oldCount * sizeof(array()->lists[0]));
            // 分类方法（二维数组的形式）添加到 methods 首部
            memcpy(array()->lists, addedLists, 
                   addedCount * sizeof(array()->lists[0]));
        }
        else if (!list  &&  addedCount == 1) {
            // 0 lists -> 1 list
            list = addedLists[0];
        } 
        else {
            // 1 list -> many lists
            List* oldList = list;
            uint32_t oldCount = oldList ? 1 : 0;
            uint32_t newCount = oldCount + addedCount;
            setArray((array_t *)malloc(array_t::byteSize(newCount)));
            array()->count = newCount;
            if (oldList) array()->lists[addedCount] = oldList;
            memcpy(array()->lists, addedLists, 
                   addedCount * sizeof(array()->lists[0]));
        }
    }

```


```
// cpp 中的函数
// 从 __src 首地址开始的 __len 长度的内存 移动到 __dst 目的地址上
void *memmove(void *__dst, const void *__src, size_t __len);
// // 从 __src 首地址开始的 __len 长度的内存复制到 __dst 目的地址上
void	*memcpy(void *__dst, const void *__src, size_t __n);
```

通过上述的源码，Category 的方法就添加到了类的 `class_rw`的` method_list `中了。而且根据源码中 `while(i--)：、memmove 、memcpy ` 可以知道越晚参与编译的分类，分类中的方法越靠前。最后参与编译的分类方法，会在方法的头部，所以，**当 Category 中定义了同主类相同方法时或多个 Category 定义了相同的方法时，最后参与编译的 Category 中的方法会被调用。**让人感觉其它相同的方法「被覆盖了」。其实只是方法查找顺序的问题，方法查找时只要找到对应名字的方法就返回了，不会继续向后寻找。

我们可以通过打印类中的方法来证明方法并直接没有覆盖。

```
// 在 test 分类中，添加 run 方法。
- (void)printMethodList:(Class)cls {
    unsigned int methodCount;
    Method *methodList = class_copyMethodList(cls, &methodCount);
    NSMutableArray *methods = [NSMutableArray array];
    for (NSInteger i = 0; i < methodCount; i++) {
        Method method = methodList[i];
        NSString *methodName = [NSString stringWithCString:sel_getName(method_getName(method))
                                                  encoding:NSUTF8StringEncoding];
        [methods addObject:methodName];
    }
    NSLog(@"%@",methods);
    free(methodList);
}

// 输出 
 (
    eat,
    run,
    run,
    test
)
```


## Category 和 Extension

- Extension 是在编译时期就确定了，属于类的一部分，extension一般用来隐藏类的私有信息，你必须有一个类的源码才能为一个类添加extension
- 从前文知道 Category 则是在运行时决定，而且 Category 的结构并不包含成员变量，所以我们不能直接通过 Category 为类添加新的成员变量。



