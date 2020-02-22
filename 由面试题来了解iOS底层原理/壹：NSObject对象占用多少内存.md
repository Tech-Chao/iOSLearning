# 壹：NSObject对象占用多少内存

> 题目：一个NSObject对象占用多少内存?


首先回答题目：

- 系统分配了16个字节给 NSObject 对象（通过 `malloc_size` 函数获得）
- 但NSObject对象内部只使用了8个字节的空间（64bit环境下，可以通过`class_getInstanceSize`函数获得）。

## 8个字节缘由

接下来我们通过 clang 将代码转换成 C/C++ 代码 结合 [「runtime 的源码」](https://opensource.apple.com/tarballs/objc4/) 一探究竟。


```
NSObject *obj = [[NSObject alloc] init];
```

通过以下的命令行可以编译生成64位架构的 iPhone C/C++ 代码。

```
xcrun  -sdk  iphoneos  clang  -arch  arm64  -rewrite-objc  main.m  -o  main.cpp
```

可以看到在同文件目录下生成了 main.cpp 文件，

在 cpp 文件中我们可以看到 NSObject 实现:

```
typedef struct objc_class *Class;

struct NSObject_IMPL {
	Class isa; // 相当于   struct objc_class *isa
};

```

NSObject 包含一个结构体的指针。 在64为的系统上指针占用 8 个字节（32位系统4个字节），所以 NSObject 在64位上占用 8 字节。


## 16字节缘由

但是在前文，我们通过 malloc_size 发现系统输出的内存大小又是 16 字节。这又是为何呢？ 

通过 objc4 源码，我们可以追踪下 NSObject 的 alloc 的过程。

```
alloc -> _objc_rootAlloc -> callAlloc -> class_createInstance -> _class_createInstanceFromZone
```

在 _class_createInstanceFromZone 中

```
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone, 
                              bool cxxConstruct = true, 
                              size_t *outAllocatedSize = nil)
{
    if (!cls) return nil;

    assert(cls->isRealized());

    // Read class's info bits all at once for performance
    bool hasCxxCtor = cls->hasCxxCtor();
    bool hasCxxDtor = cls->hasCxxDtor();
    bool fast = cls->canAllocNonpointer();

    size_t size = cls->instanceSize(extraBytes);
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    if (!zone  &&  fast) {
        obj = (id)calloc(1, size);
        if (!obj) return nil;
        obj->initInstanceIsa(cls, hasCxxDtor);
    } 
    else {
        if (zone) {
            obj = (id)malloc_zone_calloc ((malloc_zone_t *)zone, 1, size);
        } else {
            obj = (id)calloc(1, size);
        }
        if (!obj) return nil;

        // Use raw pointer isa on the assumption that they might be 
        // doing something weird with the zone or RR.
        obj->initIsa(cls);
    }

    if (cxxConstruct && hasCxxCtor) {
        obj = _objc_constructOrFree(obj, cls);
    }

    return obj;
}

```
其它代码我们暂时忽略，其中 size_t size = cls->instanceSize(extraBytes); 就是计算出需要分配的内存空间大小。

```
    size_t instanceSize(size_t extraBytes) {
        size_t size = alignedInstanceSize() + extraBytes;
        // CF requires all objects be at least 16 bytes.
        if (size < 16) size = 16;
        return size;
    }
```

可以看到。系统最少也会分配 16 字节的大小给 OC 的对象。 所以在前文中 malloc_size 的结果是 16。

## 注意

在 instanceSize  函数中 alignedInstanceSize 会对结构体 进行字节对齐的计算。

```

#ifdef __LP64__
#   define WORD_MASK 7UL
#else
#   define WORD_MASK 3UL
#end

static inline size_t word_align(size_t x) {
    return (x + WORD_MASK) & ~WORD_MASK;
}

/**

如： x =  17; 即(17 + 7 & ~7)
 
  0001 0001
+ 0000 0111
------------
  0001 1000// 24
& 1111 1000 // ~7 
------------
  0001 1000 // 24
  
实际上需要 17 字节的大小，但系统会自动字节对齐到 24 字节。
*/

```

除此之外，[「libmalloc源码」](https://opensource.apple.com/tarballs/libmalloc/) 查看 malloc_zone_calloc / calloc 源代码，会发现系统还会进行一次内存对齐的操作，以 16 字节的倍数分配内存大小。

```


#define SHIFT_NANO_QUANTUM		4
#define NANO_REGIME_QUANTA_SIZE	(1 << SHIFT_NANO_QUANTUM)	// 16

static MALLOC_INLINE size_t
segregated_size_to_fit(nanozone_t *nanozone, size_t size, size_t *pKey)
{
	size_t k, slot_bytes;
	
	// 最小 16 字节
	if (0 == size) { 
		size = NANO_REGIME_QUANTA_SIZE; // Historical behavior
	}
	// 确保内存大小为 16 的倍数
	k = (size + NANO_REGIME_QUANTA_SIZE - 1) >> SHIFT_NANO_QUANTUM; // round up and shift for number of quanta
	slot_bytes = k << SHIFT_NANO_QUANTUM;							// multiply by power of two quanta size
	*pKey = k - 1;													// Zero-based!

	return slot_bytes;
}

/**

如果size 为 24 
(24 + 16 - 1) >> 4
39 >> 4

 0010 0101 >> 4 = 0000 0010
 0000 0010 << 4 = 0010 0000 // 32
最终分配了 32 字节的内存。
*/
```

最终，iOS 分配的内存大小会是 16 的倍数大小。