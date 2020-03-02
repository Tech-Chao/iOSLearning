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
其中 `(bits & FAST_DATA_MASK)` 可以获得 `class_rw_t` 指针。

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

    const class_ro_t *ro; // 只读（在编译过程中就确定的类信息）
    
	 
    method_array_t methods;	// 存放着方法的二维数组
    property_array_t properties; // 存放属性的二维数组
    protocol_array_t protocols;	// 存放协议的二维数组

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
    method_list_t * baseMethodList; // 类初始时的方法（一维）
    protocol_list_t * baseProtocols; // 类初始时的协议（一维）
    const ivar_list_t * ivars;    	// 类初始时的成员变量（一维）

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
    // other ...
}
```
以上共同构成了 Class 在内存中的结构。

类和元类同为 `Class（struct objc_class）`类型,但两者包含的信息不同， 所以 `class_rw_t ` 同一个成员在表达类和元类时存放的值不同。如：`class_rw_t `的 `methods` 根据类对象和元类对象，分别存放着对象方法 和 类方法。如果是类特有的成员那么元类就是设置为NULL。

