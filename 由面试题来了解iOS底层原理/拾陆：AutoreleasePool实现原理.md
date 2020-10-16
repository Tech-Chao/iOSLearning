# 拾陆：AutoreleasePool实现原理

## 前言

通过前面[runloop文章](https://github.com/PhoenixiOSer/iOSLearning/blob/master/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/%E6%8B%BE%E8%82%86%EF%BC%9ARunloop%E7%B3%BB%E5%88%97_(1).md)中，我们知道在 mainRunloop 存在两个关于 `autoreleasePool` 的 `RunLoopObserver`,分别监听了 runloop 的 ①. 进入（Entry）、②. BeforeWaiting(准备进入休眠)和Exit(即将退出Loop) 


1.  进入（Entry）: 监听到进入后会调用 `_objc_autoreleasePoolPush()`函数创建自动释放池。
2.  准备进入休眠（BeforeWaiting）:调用`_objc_autoreleasePoolPop()` 和 `_objc_autoreleasePoolPush()` 释放旧池并创建新池。
3. 即将退出Loop(Exit)：调用`_objc_autoreleasePoolPop()` 来释放`_autoreleasePool`。

所以基本上我们不需要手动创建 AutoreleasePool。

## NSAutoreleasePool

在 MRC 的时代，我们通过 `NSAutoreleasePool` 类或者`@autoreleasepool`创建自动释放池子,并调用对象的`autorelease`方法，将对象放入自动释放池子中。

```
// main.m MRC环境
#import <Foundation/Foundation.h>
#import "Person.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *p = [[Person new] autorelease];
        NSLog(@"%@",p);
    }
    return 0;
}

```

通过`xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc  main.m `我们将代码编译生成 C++ 代码。

![](/Users/gaolailong/Documents/iOSLearningManual/Assets/由面试题来了解iOS底层原理/NSAutoreleasePool/autoreleasepool_main.png)

代码中的`@autoreleasepool{}`会转换成`__AtAutoreleasePool`的结构体对象。

```
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();} // 构造函数
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}  // 析构函数
  void * atautoreleasepoolobj;
};

```

在上图中`__AtAutoreleasePool __autoreleasepool;`首先会调用结构体的构造函数`objc_autoreleasePoolPush()`，将对象放入自动释放池中，再当结构体的局部对象`__autoreleasepool`离开作用域后，调用析构函数`objc_autoreleasePoolPop(atautoreleasepoolobj)`对自动释放池内的对象进行一次`release`操作。

## AutoreleasePool 结构
`autoreleasepool`通过`__AtAutoreleasePool`的构造函数、析构函数来创建和释放的了，那么`autoreleasepool`又是如何组织、存放自动释放池的对象的呢?

让我通过 obj4 的源码来一探究竟，首先我们可以在 `NSObject.mm` 找到 `objc_autoreleasePoolPush`的实现。它主要是调用了`AutoreleasePoolPage`类的类方法`push`，返回`AutoreleasePoolPage`对象。 

在`objc_autoreleasePoolPop`函数中也是对`AutoreleasePoolPage`对象进行操作。可以推断出`autoreleasepool`自定释放池对自动释放对象的管理正是通过`AutoreleasePoolPage`对象来实现的。


```
class AutoreleasePoolPage 
{

    static size_t const SIZE = 
#if PROTECT_AUTORELEASEPOOL
        PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
        PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif
    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
    // ... other methods
}
```

在 `AutoreleasePoolPage `类中定义了Page的空间大小`PAGE_MAX_SIZE = 4096 个字节`，除去上面代码中存放内部的成员变量外，所有剩下的空间都是用来存放 `autorelease` 的对象地址（8字节）。

成员变量`next`指向下一个自动释放对象地址的指针存放在当前的page的位置，而且每个 page 通过 `parent` 和 `child` 通过双向链表的形式连接。 `thread`记录当前page所在的线程，`AutoreleasePool`是与线程一一对应的.

![](/Users/gaolailong/Documents/iOSLearningManual/Assets/由面试题来了解iOS底层原理/NSAutoreleasePool/autoreleasePage.png)


自动释放的对象从上图的红色 begin 箭头处开始存放，当一个 page 存放达到上限后，会在创建一个 page 并通过`parent` 和 `child`两个指针关联这两个page对象。以此类推每个page都存放了。

## AutoreleasePool 工作流程

我们知道了 `AutoreleasePool` 两个主要的数据结构：`__AtAutoreleasePool `、`AutoreleasePoolPage `。现在来总结下 AutoreleasePool 工作的整个流程。

将最开始的代码`@autoreleasepool{}`替换成`__AtAutoreleasePool`的实现。

```

int main(int argc, const char * argv[]) {
//    @autoreleasepool {
    __AtAutoreleasePool atautoreleasepoolobj = objc_autoreleasePoolPush();
        Person *p = [[Person new] autorelease];
        NSLog(@"%@",p);
    objc_autoreleasePoolPop(atautoreleasepoolobj)
//  }
    return 0;
}
```

### objc_autoreleasePoolPush
`objc_autoreleasePoolPush()`函数：

```
#define POOL_BOUNDARY = nil
    static inline void *push() 
    {
        id *dest;
        if (DebugPoolAllocation) {
            dest = autoreleaseNewPage(POOL_BOUNDARY);
        } else {
            dest = autoreleaseFast(POOL_BOUNDARY);
        }
        assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
        return dest;
    }

```

首先会将`POOL_BOUNDARY（一个nil对象）`入栈，并返回当前压入栈中的地址值，用作 pop 时的终点。调用`autorelease`方法的对象会调用到 `AutoreleasePoolPage` 的 `autoreleaseFast` 函数，添加到 page 中，在`objc_autoreleasePoolPop`函数调用时会将`POOL_BOUNDARY`的地址值传入，用作自动释放池`pop`的停止边界。

当然系统会考虑到很多情况，比如当前的`page`情况比如是否存在、`page`是否满了。大概的过程如下：

- 如果存在page、且未满会直接通过next指针将对象地址存放在`next`中,存在page。
- 如果page 存在且满了，会创建一个新的page再压入`POOL_BOUNDARY`,再添加`autorelease`对象，然后通过`parent` 和 `child`将上下两个page关联
- 如果page都不存在那么会创建一个新的page、压入`POOL_BOUNDARY`后再添加`autorelease`对象。


### objc_autoreleasePoolPop
`objc_autoreleasePoolPop(atautoreleasepoolobj)`自动释放池的开始释放。其中`atautoreleasepoolobj`就是我们`push`时返回的边界地址(`POOL_BOUNDARY `)。`objc_autoreleasePoolPop`对调用`AutoreleasePoolPage`的`pop`方法，如下。

```
// token == atautoreleasepoolobj  == 每次 push 时的`POOL_BOUNDARY `
  static inline void pop(void *token) 
    {
        AutoreleasePoolPage *page;
        id *stop;

        if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
            // Popping the top-level placeholder pool.
            if (hotPage()) {
                // Pool was used. Pop its contents normally.
                // Pool pages remain allocated for re-use as usual.
                pop(coldPage()->begin());
            } else {
                // Pool was never used. Clear the placeholder.
                setHotPage(nil);
            }
            return;
        }

        page = pageForPointer(token);
        stop = (id *)token;
        if (*stop != POOL_BOUNDARY) {
            if (stop == page->begin()  &&  !page->parent) {
                // Start of coldest page may correctly not be POOL_BOUNDARY:
                // 1. top-level pool is popped, leaving the cold page in place
                // 2. an object is autoreleased with no pool
            } else {
                // Error. For bincompat purposes this is not 
                // fatal in executables built with old SDKs.
                return badPop(token);
            }
        }

        if (PrintPoolHiwat) printHiwat();

        page->releaseUntil(stop);

        // memory: delete empty children
        if (DebugPoolAllocation  &&  page->empty()) {
            // special case: delete everything during page-per-pool debugging
            AutoreleasePoolPage *parent = page->parent;
            page->kill();
            setHotPage(parent);
        } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
            // special case: delete everything for pop(top) 
            // when debugging missing autorelease pools
            page->kill();
            setHotPage(nil);
        } 
        else if (page->child) {
            // hysteresis: keep one empty child if page is more than half full
            if (page->lessThanHalfFull()) {
                page->child->kill();
            }
            else if (page->child->child) {
                page->child->child->kill();
            }
        }
    }
```
主要是三个操作

- 使用 pageForPointer 获取当前 token 所在的 page,
- 调用 releaseUntil 方法释放栈中的对象，直到 stop 边界（即push时返回的`POOL_BOUNDARY`地址）。
- 调用 child 的 kill 方法，删除空的`child` page 对象。

## 延伸1： AutoReleasePool的循环嵌套

有时候我们会碰到`AutoReleasePool`嵌套的情况，那么自动释放的对象又是如何组织和管理的呢？

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSObject *obj1 = [[NSObject alloc] init];
        
    @autoreleasepool {
        NSObject *obj2 = [[NSObject alloc] init];
        NSObject *obj3 = [[NSObject alloc] init];
        
          @autoreleasepool {
              NSObject *obj4 = [[NSObject alloc] init];
            }
      }
    }
    return 0;
}
```

按照我们之前对 `autoreleasepool`的理解后，我们可以将代码转换成：

```
int main(int argc, const char * argv[]) {
     atautoreleasepoolobj1 = objc_autoreleasePoolPush();
        NSObject *obj1 = [[NSObject alloc] init];
    	 atautoreleasepoolobj2 = objc_autoreleasePoolPush();
        NSObject *obj2 = [[NSObject alloc] init];
        NSObject *obj3 = [[NSObject alloc] init];
           atautoreleasepoolobj3 = objc_autoreleasePoolPush();
              NSObject *obj4 = [[NSObject alloc] init];
            objc_autoreleasePoolPop(atautoreleasepoolobj3)
      objc_autoreleasePoolPop(atautoreleasepoolobj2)
    objc_autoreleasePoolPop(atautoreleasepoolobj1)
    return 0;
}
```
 

每层的嵌套，都会调用一次`objc_autoreleasePoolPush`函数，在`objc_autoreleasePoolPush`时会有一次入栈操作（`POOL_BOUNDARY`入栈）。作为这一层`autoreleasepool`pop的停止边界，大概的结构如下图。

![](/Users/gaolailong/Documents/iOSLearningManual/Assets/由面试题来了解iOS底层原理/NSAutoreleasePool/nest_autoreleasepool.png)




## 延伸2： Runloop 与AutoReleasePool的关系
注意：在上文提到过在`mainRunloop`中存在`RunLoopObserver`,分别监听了 runloop 的 ①. 进入（Entry）、②. BeforeWaiting(准备进入休眠)和Exit(即将退出Loop),会自动调用`_objc_autoreleasePoolPush()` 、`_objc_autoreleasePoolPop()` 所以一般情况下我们可以不用自己手动的去创建`autoreleasePool`

## 延伸3： autorelease对象在什么时候释放

autorelease对象的释放在不同场景下释放的时机不一样，

- autorelease对象直接是由`@autorelasepool{}`包裹，那么autorelease对象会在`@autorelasepool{}`的大括号之后释放。
- 没有显式使用`@autorelasepool{}`，会根据某次Runloop循环中调用`_objc_autoreleasePoolPop()`释放（可能是在runloop休眠之前或者退出runloop时）。

tip: ARC模式下，方法内的局部变量会在方法的大括号之后就会被释放，ARC模式下可能直接插入的 release 方法。



