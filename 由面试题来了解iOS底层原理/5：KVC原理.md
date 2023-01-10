# 伍：KVC原理

> 通过KVC修改属性会触发KVO么？
> KVC的赋值和取值过程是怎样的？原理是什么？答：
KVC修改属性会触发KVO。过程参见下文

## KVC使用

KVC的全称是 Key-Value Coding，俗称“键值编码”，通过一个key来访问、设置属性。

```
// 常用Api
- (void)setValue:(id)value forKeyPath:(NSString *)keyPath;- (void)setValue:(id)value forKey:(NSString *)key;- (id)valueForKeyPath:(NSString *)keyPath;- (id)valueForKey:(NSString *)key; 
- ..
```

代码的使用

```
- (void)viewDidLoad {
	[super viewDidLoad];
    Person *p = [[Person alloc] init];
    [p addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionOld | NSKeyValueObservingOptionNew context:nil];
    [p setValue:@1 forKey:@"age"];
	//[p setValue:@1 forKeyPath:@"age"]; // forKeyPath 可以深层次的去寻找我们需要的属性。
    NSLog(@"age is :%d",p.age);
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    NSLog(@"监听到%@的%@发生了改变%@",object,keyPath,change);
}


// 输出
监听到<Person: 0x600003720130>的age发生了改变{
    kind = 1;
    new = 1;
    old = 0;
}

age is :1
```
通过代码，可以看到 KVC 确实可以触发 KVO。那么 KVC 又是如何触发的呢？


## KVC setValue:forkey:实现原理

```
@interface Person : NSObject
{
    @public
    int _age;
    int _isAge;
    int age;
    int isAge;
}
//@property (nonatomic, assign) int age;
@end

// key: age
[p setValue:@1 forKey:@"age"];

// 上诉可以通过注释和log。观察赋值的顺序
```

- 首先会在类对象中按顺序寻找 `set<Key>:`、`_set<Key>:`两个方法。
- 如果以上两个方法都不存在会根据`+ (BOOL)accessInstanceVariablesDirectly` 返回值做处理（默认返回YES）。
	- 如果`accessInstanceVariablesDirectly `方法返回`NO`,调用`setValue:forUndefinedKey:`并抛出异常`NSUnknownKeyException`	- 如果返回`YES`,会直接给成员变量 Key 赋值，按照`_<key>、_is<key>、<key>、is<Key>`的顺序赋值。最终如果未找到上述的任何一个成员变量，调用`setValue:forUndefinedKey:`并抛出异常`NSUnknownKeyException`

![](https://github.com/PhoenixiOSer/iOSLearningManual/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/KVO%20setter.png?raw=true)

## KVC valueForKey：实现

```
@implementation Person
//get<Key>, <key>, is<Key>, or _<key>,
- (NSString *)getAge{
    return NSStringFromSelector(_cmd);
}
- (NSString *)age{
    return NSStringFromSelector(_cmd);
}
- (NSString *)isAge{
    return NSStringFromSelector(_cmd);
}
- (NSString *)_age{
    return NSStringFromSelector(_cmd);
}
@end

Person *person = [[Person alloc] init];
NSLog(@"取值:%@",[person valueForKey:@"age"]);
```

- 首先会先去按顺序查找 `get<Key>, <key>, is<Key> 或 _<key> `方法。
- 如果以上两个方法都不存在会根据`+ (BOOL)accessInstanceVariablesDirectly` 返回值做处理（默认返回YES）。
 	- 如果`accessInstanceVariablesDirectly `方法返回`NO`,调用`valueForUndefinedKey:` 方法,抛出Error `'NSUnknownKeyException'。
	- 返回 YES，则按顺序搜索名为 `_<key>、_is<key>、<key>、is<Key>` 的实例变量，如果找到，则直接获取实例变量的值并将值转换成相应类型返回。如果未如找到上述的任何一个成员变量，调用 `valueForUndefinedKey:` 方法,抛出Error `'NSUnknownKeyException'。

![](https://github.com/PhoenixiOSer/iOSLearningManual/blob/master/Assets/%E7%94%B1%E9%9D%A2%E8%AF%95%E9%A2%98%E6%9D%A5%E4%BA%86%E8%A7%A3iOS%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/KVO%20getter.png?raw=true)

## KVC 补充

### KVC Collection Operators(KVC 集合运算)

对于数组来说，我们可以通过 KVC 来获取数组某些字段的和、平均值、最大值、最小值。valueForKeyPath 设置 KeyPath 分别为 @sum @avg @max @min`。


```
@interface Product : NSObject
@property NSString *name;
@property double price;
@property NSDate *launchedOn;
@end

// Product *p1 p2 p3 p4
NSArray *products = @[p1,p2,p3,p4];

[products valueForKeyPath:@"@count"]; // 4
[products valueForKeyPath:@"@sum.price"]; // 3526.00
[products valueForKeyPath:@"@avg.price"]; // 881.50
[products valueForKeyPath:@"@max.price"]; // 1699.00
[products valueForKeyPath:@"@min.launchedOn"]; // June 11, 2012
```

更多相关内容：[kvc-collection-operators](https://nshipster.com/kvc-collection-operators/)
