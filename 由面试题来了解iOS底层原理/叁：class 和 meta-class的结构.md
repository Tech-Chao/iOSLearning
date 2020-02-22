# 叁：class 和 meta-class的结构


class 和 meta-class 都是 Class 类型。在 obj4 源码中，Class 是指向 objc_class 结构体的指针。

```
typedef struct objc_class *Class;
```

从 __OBJC2__ 开始: 

```

struct objc_object {
private:
    isa_t isa;
...
}

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // 方法缓存
    class_data_bits_t bits;    // 类的具体信息
    
   // other methods ... 
}
```
其中 `(bits & FAST_DATA_MASK)` 可以获得 `class_rw_t`。

```
	class_rw_t* data() {
        return (class_rw_t *)(bits & FAST_DATA_MASK);
    }
```
`class_rw_t` 包含了 方法信息、属性信息、协议信息

```
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;
    // other methods ..
 }


struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
    // other ...
}
```
以上共同构成了 Class 在内存中的结构。

类对象和元类对象共用 `Class（struct objc_class）`,所以 `struct objc_class ` 在不同的对象类型下可能存放不同的值，或者是为空。如：methods 根据类对象和元类对象，分别存放着对象方法 和 类方法。