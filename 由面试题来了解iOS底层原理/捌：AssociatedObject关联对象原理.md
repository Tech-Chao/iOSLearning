# 捌：AssociatedObject关联对象原理


在 Category 讨论的文章中，我们可以使用关联对象的技术来给 Category 添加属性，那么关联对象又是如何添加的呢？本篇文章给出了关联对象的原理解读。


在 Category 中使用`@property`声明的属性，只会生成相关的方法声明，并不会生成相应的成员变量和方法的实现：

```
@interface Person (Test)

@property (nonatomic, assign) int weight;
/**
等价于
- (void)setWeight:(int)weight;
- (int)weight;
*/

@end
```


## 关联对象的使用

```
@interface Person (Test)
@property (nonatomic, copy) NSString *name;

@end

#import <objc/message.h>

static const char *nameKey;

@implementation Person (Eat)

- (void)setName:(NSString *)name{
    objc_setAssociatedObject(self, &nameKey, name, OBJC_ASSOCIATION_COPY);
    
}

- (NSString *)name {
    objc_getAssociatedObject(self,&nameKey)
}

@end

```

其中 `OBJC_ASSOCIATION_COPY` 是关联对象使用的策略：

```
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           // 相当于 assgin
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, // strong,nonatomic
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   // copy,nonatomic
    OBJC_ASSOCIATION_RETAIN = 01401,       // strong,atomic
    OBJC_ASSOCIATION_COPY = 01403          // copy,atomic
};
```

`<key>`可以使用多种方式表达:

```
推荐使用的<key>的方式：
1.  static const char *nameKey; &nameKey
2. 使用@selector(name)
```

## 关联对象的原理

关联对象能够实现的核心对象：

```
AssociationsManagerAssociationsHashMapObjectAssociationMapObjcAssociation
```


我们可以在 objc4 的源码中在`objc-references.mm`中找到上面的4个对象。

下面是简化过的代码：
![](/Users/gaolailong/Documents/iOSLearningManual/Assets/由面试题来了解iOS底层原理/associatedObject@2x.png)

可以总结如下：
- 存在一个全局统一的 `AssociationsManager` 对象，AssociationsManager 包含一个 `AssociationsHashMap`。
- `AssociationsHashMap`以`disguised_ptr_t`为key, `ObjectAssociationMap`为value。
- `ObjectAssociationMap` 又以`void *`为key，`ObjcAssociation`为Value。
- `ObjcAssociation` 中包含了 `_poilcy` 和 我们的关联对象的值`Value`。

所以我们可以认为所有的关联对象都是存放在`AssociationsManager` 中。

![](/Users/gaolailong/Documents/iOSLearningManual/Assets/由面试题来了解iOS底层原理/associated_imp.png)

### 关联对象 set
调用关联对象设置值的时候`objc_setAssociatedObject(id _Nonnull object, const void * _Nonnull key,id _Nullable value, objc_AssociationPolicy policy)`，最终系统会调用：

```
void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {

	// 保护判断
    if (!object && !value) return;
    assert(object);
    if (object->getIsa()->forbidsAssociatedObjects())
        _objc_fatal("objc_setAssociatedObject called on instance (%p) of class %s which does not allow associated objects", object, object_getClassName(object));
        
   
    ObjcAssociation old_association(0, nil);
    id new_value = value ? acquireValue(value, policy) : nil;
    {
     	 //  获取全局统一的 AssociationsManager 和 AssociationsHashMap
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
		// 用 object 通过`DISGUISE`函数生成key: disguised_ptr_t
        disguised_ptr_t disguised_object = DISGUISE(object);
        if (new_value) {
            通过 key disguised_object找到对应的 ObjectAssociationMap。
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i != associations.end()) {
                // secondary table exists
                ObjectAssociationMap *refs = i->second;
                // 找到对应对象的 ObjectAssociationMap 后，用 key 作为 value
                ObjectAssociationMap::iterator j = refs->find(key);
					// 用policy、 new_value 生成一个ObjcAssociation，覆盖或新增
                if (j != refs->end()) {
                    old_association = j->second;
                    j->second = ObjcAssociation(policy, new_value);
                } else {
                    (*refs)[key] = ObjcAssociation(policy, new_value);
                }
            } else {
            		// 首次使用 object 作为key，创建新的 ObjectAssociationMap
                ObjectAssociationMap *refs = new ObjectAssociationMap;
                associations[disguised_object] = refs;
                (*refs)[key] = ObjcAssociation(policy, new_value);
                object->setHasAssociatedObjects();
            }
        } else {
            // 如果 new_value 是为nil 擦除原来的关联对象。
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i !=  associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    refs->erase(j);
                }
            }
        }
    }
    // release the old value (outside of the lock).
    if (old_association.hasValue()) ReleaseValue()(old_association);
}

```

同样的使用`objc_getAssociatedObject(id _Nonnull object, const void * _Nonnull key)` 时系统会调用`id _object_get_associative_reference(id object, void *key)`函数。源码和`_object_set_associative_reference `类似就不再粘贴出来了，有兴趣的小伙伴可以去 objc4 的源码中的`objc-references.mm`自行查阅。