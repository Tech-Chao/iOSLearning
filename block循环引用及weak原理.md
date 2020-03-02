# 拾壹：block循环引用及weak原理

## 面试题
```
1. 使用 block 的注意事项?如何解决循环引用?
2. weak指针在对象销毁后指向 nil 的实现原理
```

第一题：
答： block 使用主要关注内存是否泄漏，block 容易造成循环引用，解决循环引用主要有两种：

  1. 使用 `__weak`（推荐）、`__unsafe_retained`(不推荐)修饰被 block 的捕获的变量。
  2. 在block执行的代码块中，将捕获的变量重置为 nil，缺点是必须执行完 block 块才会解决循环引用。

第二题：
答：当变量被修饰为 weak 时候，weak 变量会被放入到 weak表中，


## weak原理
查看 objc4 的源码 `NSObject.mm` 中，存在 `objc_initWeak(id *location, id newObj)`函数，当有 `__weak`修饰变量的时候就会被调用。

```
// 1.初始化一个指向 nil 的 weak 指针 
__weak id weakPtr;

// 2.初始化一个 weak 指针，指向 NSObject 对象。
 NSObject *o = [[NSObject alloc] init];
 __weak id weakPtr = o;
```

weak 指针初始化步骤：

1. objc_initWeak
2. storeWeak

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/__weak.png?raw=true)

首先判断需要被指向的对象是否存在，如果不存在直接返回 nil。存在则调用`storeWeak `函数，其中参数 `location`是 `__weak`指针的地址，`newObj`是指向对象的指针。


```
static id storeWeak(id *location, objc_object *newObj)
{
    assert(haveOld  ||  haveNew);
    if (!haveNew) assert(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    SideTable *oldTable;
    SideTable *newTable;

    // Acquire locks for old and new values.
    // Order by lock address to prevent lock ordering problems. 
    // Retry if the old value changes underneath us.
 retry:
    if (haveOld) {
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (haveNew) {
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }

    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);

    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    // Prevent a deadlock between the weak reference machinery
    // and the +initialize machinery by ensuring that no 
    // weakly-referenced object has an un-+initialized isa.
    if (haveNew  &&  newObj) {
        Class cls = newObj->getIsa();
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) 
        {
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
            class_initialize(cls, (id)newObj);

            // If this class is finished with +initialize then we're good.
            // If this class is still running +initialize on this thread 
            // (i.e. +initialize called storeWeak on an instance of itself)
            // then we may proceed but it will appear initializing and 
            // not yet initialized to the check above.
            // Instead set previouslyInitializedClass to recognize it on retry.
            previouslyInitializedClass = cls;

            goto retry;
        }
    }

    // Clean up old value, if any.
    if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // Assign new value, if any.
    if (haveNew) {
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected

        // Set is-weakly-referenced bit in refcount table.
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();
        }

        // Do not set *location anywhere else. That would introduce a race.
        *location = (id)newObj;
    }
    else {
        // No new value. The storage is not changed.
    }
    
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    return (id)newObj;
}
```

其中 `SideTable` 是一个结构体：

```
struct SideTable {
	// 自旋锁
    spinlock_t slock;
    // 引用计算
    RefcountMap refcnts;
    /// weak 表
    weak_table_t weak_table;

    // other ...
}
```
