# 拾伍：weak指针工作原理

## 面试题

>
>1. weak指针的工作原理？对象没有强引用时，如何实现将指向当前对象的弱指针置为nil。
>
>答：当变量被修饰为 weak 时候，weak 变量会被放入到 weak 表（散列表）中，当weak指针指向的对象引用计数为0的时候，`dealloc`方法会调用到`clearDeallocating `函数，从`SideTable` 中`weak_table`以对象作为 key 获取所有的弱引用并置为nil。详见下文。
>
>


## weak修饰
先来看一段简单的代码：

```
  NSLog(@"Berfore");
  {
      Person *p1 = [[Person alloc] init];
      __weak Person *weakP = p1;
  }
  NSLog(@"After");
```

生成一个 p1 的 Person 实例对象，再创建一个 weakP 的弱指针，调试代码时在 Debug 中打开`Always show Disassembly`，值得我们关注的已经用红框标识。

![](/Users/gaolailong/Documents/iOSLearningManual/Assets/由面试题来了解iOS底层原理/weak指针/objc_initWeak.png)

## weak指针的存储

在汇编代码中可以看到系统调用了`objc_initWeak`, 通过在 obj4 的源码中搜索可以看到在 `NSObject.mm` 中，存在 `objc_initWeak(id *location, id newObj)`函数。

![](https://github.com/PhoenixiOSer/iOSLearning/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/__weak.png?raw=true)


参数 `location`是 `__weak`指针的地址，`newObj`是对象指针。首先判断需要被指向的对象是否存在，如果不存在直接返回 nil。存在则调用`storeWeak `函数，`DontHaveOld、DoHaveNew`是个枚举值相当于false、true之类。


```
enum HaveOld { DontHaveOld = false, DoHaveOld = true };
enum HaveNew { DontHaveNew = false, DoHaveNew = true };


template <HaveOld haveOld, HaveNew haveNew,
          CrashIfDeallocating crashIfDeallocating>
static id storeWeak(id *location, objc_object *newObj)
{
    assert(haveOld  ||  haveNew);
    if (!haveNew) assert(newObj == nil);
    
    Class previouslyInitializedClass = nil;
    id oldObj;
	// 声明两个SideTable
    SideTable *oldTable;
    SideTable *newTable;

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
	// location 应该与 oldObj 保持一致，如果不同，说明当前的 location 已被修改，重新获取 oldTable
    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

	 
    // 为防止弱引用和+initialize之间的死锁
    // 下面代码确保弱引用指针指向的对象是一个初始化好的类
    if (haveNew  &&  newObj) {
        Class cls = newObj->getIsa();
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) 
        {
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
            // 调用初始化 initialize 方法
            class_initialize(cls, (id)newObj);
            // 记录当亲初始化的类，重新开始检测
            previouslyInitializedClass = cls;

            goto retry;
        }
    }
    // 清除可能存在的旧值
    if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

	// 分配新值
    if (haveNew) {
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating);

        // Set is-weakly-referenced bit in refcount table.
        // 标识 refcount 表中的 weak 引用位 为yes
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();
        }
        *location = (id)newObj;
    }
    else {
    }
    
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    return (id)newObj;
}
```

上面源码中有三个数据结构需要我们注意：`SideTables`、`SideTable`、`weak_table_t`。

三者的关系如图：

![](/Users/gaolailong/Documents/iOSLearningManual/Assets/由面试题来了解iOS底层原理/weak指针/sideTables.png)

在了解 weak 工作原理之前我们需要对这三个数据结构要有大概的认识。

### SideTables

`SideTables`是一个全局的Hash表。用来管理所有对象的引用计数和weak指针，`SideTables`使用对象的内存地址当它的`key`。value 则是`SideTable`的结构体。

```
// SideTables() 返回的是 StripedMap<SideTable>&。

template<typename T>
class StripedMap {
// `SideTables` 在非模拟器的 iphone 上的容量大小是 8 。
#if TARGET_OS_IPHONE && !TARGET_OS_SIMULATOR
    enum { StripeCount = 8 };
#else
    enum { StripeCount = 64 };
#endif

    struct PaddedT {
        T value alignas(CacheLineSize);
    };

    PaddedT array[StripeCount];
	
	// hash 表的查找算法
    static unsigned int indexForPointer(const void *p) {
        uintptr_t addr = reinterpret_cast<uintptr_t>(p);
        return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
    }

	// ...
}
```

可以看到在真机的 iPhone 设备上 `SideTables`的容量为8，且hash表的查找方法：地址指针右移4位异或地址指针右移9位，对StripeCount取余。

### SideTable

`SideTable` 是一个结构体,包含一个自旋锁、`RefcountMap`引用计数表、`weak_table_t`弱引用表。

```
struct SideTable {
	// 自旋锁
    spinlock_t slock;
    // 引用计数表
    RefcountMap refcnts;
    /// weak 表
    weak_table_t weak_table;

    // other ...
}
```

在[拾壹：Runtime系列_(1)](https://github.com/PhoenixiOSer/iOSLearning/blob/master/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/%E6%8B%BE%E5%A3%B9%EF%BC%9ARuntime%E7%B3%BB%E5%88%97_(1).md)中我们知道在 arm64 之后 isa 指针是被优化过，在 isa 的64位中存放着很多有效信息，其中对象的引用计数会被存放在 19 位的`extra_rc`中，而如果引用计数过大则会设置 isa 的`has_sidetable_rc`位为1，将引用计数存放在`SideTable`的`RefcountMap`中。

#### RefcountMap

`RefcountMap` 是一个 C++ 的map，通过对象指针寻找对应的引用计数（`table.refcnts.find(this)`）。

![](/Users/gaolailong/Documents/iOSLearningManual/Assets/由面试题来了解iOS底层原理/weak指针/RefcountMap.png)

`RefcountMap `存放的是个8字节的一个无符号整型，其结构如上图中的右部分。

```
#define SIDE_TABLE_WEAKLY_REFERENCED (1UL<<0)  标识该对象是否有过 weak 对象；
#define SIDE_TABLE_DEALLOCATING      (1UL<<1)  标识该对象是否正在 dealloc
#define SIDE_TABLE_RC_ONE            (1UL<<2)  从这个位置开始存放引用计数数值
```


### `weak_table_t`

![](/Users/gaolailong/Documents/iOSLearningManual/Assets/由面试题来了解iOS底层原理/weak指针/weak_table_t.png)

`weak_table_t`是全局的弱引用表，会将被弱引用的对象做为key，`weak_entries`作为value。

```
struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[4];
        };
    };
	// ...
}
```

`weak_entries`又包含了对象 `referent` 及所有指向该对象的弱引用指针地址数组`referrers`。 

- 注意`inline_referrers`也是存放指向该对象的弱引用指针地址的数组，当弱引用大于4个的时候才会使用`referrers`存放。


接下来让我们跳到上面代码「分配新值部分」系统调用`weak_register_no_lock `传入 `weak_table `、`被weak引用的对象(newObj)`、`weak指针的地址(location)`。

```
id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
                      id *referrer_id, bool crashIfDeallocating)
{
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    // 如果被引用的对象为nil 或者被引用的对象是 TaggedPointer 返回对象本身
    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    // 确保被引用的对象正在被销毁 deallocating
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating();
    }
    else {
        BOOL (*allowsWeakReference)(objc_object *, SEL) = 
            (BOOL(*)(objc_object *, SEL))
            object_getMethodImplementation((id)referent, 
                                           SEL_allowsWeakReference);
        if ((IMP)allowsWeakReference == _objc_msgForward) {
            return nil;
        }
        deallocating =
            ! (*allowsWeakReference)(referent, SEL_allowsWeakReference);
    }

    if (deallocating) {
        if (crashIfDeallocating) {
            _objc_fatal("Cannot form weak reference to instance (%p) of "
                        "class %s. It is possible that this object was "
                        "over-released, or is in the process of deallocation.",
                        (void*)referent, object_getClassName((id)referent));
        } else {
            return nil;
        }
    }

    // 以上属于前期保护判断,现在开始存储弱引用指针
    weak_entry_t *entry;
    // 如果对象本来就存在弱引用，weak_entry_t 上添加弱引用指针的地址 referrer
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        append_referrer(entry, referrer);
    } 
    else {
        //  首次被弱引用，新建weak_entry_t
        weak_entry_t new_entry(referent, referrer);
        // weak_table 是否需要扩容（占用超过3/4容量时）
        weak_grow_maybe(weak_table);
        //
        weak_entry_insert(weak_table, &new_entry);
    }
    return referent_id;
}

```

其中 weak_table 的 hash 查找算法是:

```
static inline uintptr_t hash_pointer(objc_object *key) {
    return ptr_hash((uintptr_t)key);
}

static inline uint32_t ptr_hash(uint64_t key)
{
    key ^= key >> 4;
    key *= 0x8a970be7488fda55;
    key ^= __builtin_bswap64(key);
    return (uint32_t)key;
}
```

至此，weak指针就被转换成`weak_entry_t`的结构体存放在了`SideTable`中的`weak_table`中。

## weak指针置nil

当我们知道了 weak 指针是如何存储的话，那么销毁也就是一个差不多的过程。通过被弱引用的对象获取到`SideTable`,在通过被弱引用的对象从 `SideTable`中获取`weak_table`。在根据被弱引用对象从`weak_table `获取之前存放的`weak_entry_t `。将其的`referrers/inline_referrers` 遍历置 `*referrer` 为nil即可。
下面直接通过 objc4 的源码大致的过一遍。

在`NSObject.mm`的 dealloc 的过程：

- `dealloc`
- `_objc_rootDealloc`
- `rootDealloc`
- `object_dispose`
- `objc_destructInstance`
- `clearDeallocating_slow`
- `weak_clear_no_lock`

```
void 
weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
  objc_object *referent = (objc_object *)referent_id;
    // 根据被弱引用对象获取 weak_entry_t
    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {

        return;
    }

    weak_referrer_t *referrers;
    size_t count;
    // 判断弱引用真实存放位置
    if (entry->out_of_line()) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } 
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    // 遍历循环将 *referrer 置为nil
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
            else if (*referrer) {
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n", 
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    // 从 weak_table 移除该弱引用weak_entry_t
    weak_entry_remove(weak_table, entry);
```

