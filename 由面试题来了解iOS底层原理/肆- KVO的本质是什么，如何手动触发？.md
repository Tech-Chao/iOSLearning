# 肆: KVO的本质是什么，如何手动触发？

> 问题：KVO的本质是什么，如何手动触发？

答：

- KVO会利用RuntimeAPI动态生成一个被监听对象的子类，并且让该实例对象的isa指向这个全新的子类- 当修改实例对象的属性时，会调用Foundation框架的_NSSetXXXValueAndNotify函数:	1. willChangeValueForKey:	2. 父类原来的setter	3. didChangeValueForKey:	4. 内部会触发监听器（Oberser）的监听方法( observeValueForKeyPath:ofObject:change:context:）

如何手动触发?:手动调用willChangeValueForKey:和didChangeValueForKey:## KVO原理

首先来看一段使用 KOV 的代码:

``` 
	#import <objc/message.h>

	Person *p1 = [[Person alloc] init];
	Person *p2 = [[Person alloc] init];
	NSLog(@"KVO 之前的isa指针地址：%p %p",object_getClass(p1),object_getClass(p2));
    NSLog(@"KVO 之前的类对象：%@ %@",object_getClass(p1),object_getClass(p2));
    [p2 addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionOld | NSKeyValueObservingOptionNew context:nil];
    NSLog(@"KVO 之后的isa指针地址：%p %p",object_getClass(p1),object_getClass(p2));
     NSLog(@"KVO 之后的类对象：%@ %@",object_getClass(p1),object_getClass(p2));
    
    
//    输出
KVO 之前的isa指针地址：0x109f46e80 0x109f46e80
KVO 之前的isa指针地址：Person Person

KVO 之后的isa指针地址：0x109f46e80 0x6000017deeb0
KVO 之后的isa指针地址：Person NSKVONotifying_Person
    
```

可以看到使用 KVO 后 Person 的实例对象的 isa 指向的地址（即指向的类对象）是不同的。上面输出的类名证明了 KVO 会动态生成一个子类`NSKVONotifying_Person`。KVO 之所以能够实现属性的监听和这个`NSKVONotifying_Person`息息相关。

### NSKVONotifying_Person的内部结构

Person 的内部主要结构的如下：

![](https://github.com/PhoenixiOSer/iOSLearningManual/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/Person%E7%BB%93%E6%9E%84.png?raw=true)

在 Person 中，存在 isa 指针指向元类、superClas 指向父类（NSObject）、对象方法 `setAge：、age`等。

 
而 NSKVONotifying_Person 内部包含及多少个对象方法。我们可以通过 runtime 来查看。

```
[self printMethodNamesOfClass:object_getClass(p2)];

- (void)printMethodNamesOfClass:(Class)cls {
    unsigned int count;
    Method *methodList = class_copyMethodList(cls, &count);
    for (int i = 0; i < count; i++) {
        Method method = methodList[i];
        SEL selector =  method_getName(method);
        NSLog(@"%@",NSStringFromSelector(selector));
    }
    free(methodList);
}

输出：
setAge:
class
dealloc
_isKVOA
```

可以看出 `NSKVONotifying_Person` 会对这四个方法进行重写。

- `- setAge:`的方法实现指向了Foundation框架的_NSSetXXXValueAndNotify函数。
- `- class` 重写返回父类的类型 `Person`，对外封装具体的实现;
- `- _isKOV` 返回YES
- `- dealloc`在对象被销毁会进行 KVO 对象的处理
- `NSKVONotifying_Person` 类对象是继承自 Person 的。所以 superClass 指向 Person。
- isa 会指向 `NSKVONotifying_Person `自己的元类。


![](https://github.com/PhoenixiOSer/iOSLearningManual/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/NSKVONotifying_Person%E7%BB%93%E6%9E%84.png?raw=true)


### NSKVONotifying_Person 中的 setAge: 方法

```
NSLog(@"KVO 之前的方法地址：%p %p", [p1 methodForSelector:@selector(setAge:)], [p2 methodForSelector:@selector(setAge:)]);
[p2 addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionOld | NSKeyValueObservingOptionNew context:nil];    
NSLog(@"KVO 之后的方法地址：%p %p", [p1 methodForSelector:@selector(setAge:)], [p2 methodForSelector:@selector(setAge:)]);
```

![](https://github.com/PhoenixiOSer/iOSLearningManual/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/console%20print.png?raw=true)

通过在 console 中 p 方法实现可以看到从原先的 `Person setAge:`变成调用 `Foundation` 框架的 `_NSSetIntValueAndNotify`函数(注意`_NSSetIntValueAndNotify `中的 Int 会根据被监听的属性类型不同而改变，如：`_NSSetDoubleValueAndNotify`)。


```
// 伪代码
- (void)setAge:(int)age
{
 _NSSetIntValueAndNotify();
}

void _NSSetIntValueAndNotify() {
{
	[self willChangeValueForKey:@"age"];
	[super setAge:age];
	[self didChangeValueForKey:@"age"];
}
```

## 手动触发 KVO

通过重写父类 Person 的willChangeValueForKey:和didChangeValueForKey:方法，模拟他们的实现。

```
- (void)setAge:(int)age
{
    NSLog(@"setAge:");
    _age = age;
}
- (void)willChangeValueForKey:(NSString *)key
{
    NSLog(@"willChangeValueForKey: - begin");
    [super willChangeValueForKey:key];
    NSLog(@"willChangeValueForKey: - end");
}
- (void)didChangeValueForKey:(NSString *)key
{
    NSLog(@"didChangeValueForKey: - begin");
    [super didChangeValueForKey:key];
    NSLog(@"didChangeValueForKey: - end");
}


// 输出：
willChangeValueForKey: - begin
willChangeValueForKey: - end
setAge:
didChangeValueForKey: - begin
监听到<Person: 0x6000021c0770>的age发生了改变{
    kind = 1;
    new = 1;
    old = 2;
}
didChangeValueForKey: - end
```

发现设置 age 属性时，`willChangeValueForKey、didChangeValueForKey`会依次被调用。并且监听的动作是在`didChangeValueForKey`中被触发的。所以猜测我们可以通过`willChangeValueForKey 、didChangeValueForKey`来手动触发 KVO。


```
Person *p = [[Person alloc] init];
p.age = 1;
   
NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
[p addObserver:self forKeyPath:@"age" options:options context:nil];
    
    
[p willChangeValueForKey:@"age"]; // willChangeValueForKey 不可缺少，不然不会成功出发 KVO。
[p didChangeValueForKey:@"age"];

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
	NSLog(@"监听到%@的%@发生了改变%@",object,keyPath,change);
}


// 输出：
监听到<Person: 0x6000021c0770>的age发生了改变{
    kind = 1;
    new = 1;
    old = 1;
}
   
```

输出中可以看到，我们并没有对 age 的值并没有发生变化，`new 、old` 都为 1，但是系统还是调用了`observeValueForKeyPath:ofObject:change:context:`





