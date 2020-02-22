# 贰：对象的isa指针指向哪里？

> 题目：对象的isa指针指向哪里?

答： 根据对象的类型不同，isa 指针指向也不同：

- instance对象的isa指向class对象- class对象的isa指向meta-class对象- meta-class对象的isa指向基类的meta-class


## 实例对象

```
NSObject *obj = [[NSObject alloc] init] ;

/**
obj 就称之为 实例对象（instance对象），存放了 isa 指针。
*/
```
实例对象存储的信息主要包括：

- isa 变量
- 其它成员变量

## 类对象

```
Class  objectClass1 = [obj class];
Class  objectClass2 = [NSObject class];

/**
objectClass1、objectClass2地址值相等，指向的是同一个类对象（Class 对象）
*/
```

class对象在内存中存储的信息主要包括：

- isa 指针
- superclass 指针
- 类的属性信息（property）、类的对象方法信息（instance methods）
- 类的协议信息 （protocol）、类的成员变量信息（Ivar）
- ...


## 元类对象


```
Class objectMetaClass = object_getClass([NSObject class]);

Class objectMetaClass = object_getClass(objectClass2);

```
metaClass 对象在内存中存储的信息主要包括：

- isa指针- superclass指针- 类的类方法信息（class method）


## 总结

- 实例对象的 isa "指向" 类对象。
- 类对象的 isa "指向" 元类对象。
- 元类对象的 isa "指向" 根元类对象。

![isa](https://github.com/PhoenixiOSer/iOSLearningManual/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/isa%20%E6%8C%87%E9%92%88.png?raw=true)


注意：从64位架构之后，"指向"并不是真正的指向，而是会通过 isa 指针指向的内存地址按位与（&）上 ISA_MASK(0x0000000ffffffff8ULL) ,得出真正的指向地址。

```
# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL
# elif __x86_64__
#   define ISA_MASK        0x00007ffffffffff8ULL
# endif
```