# iOS事件处理中响应链
> 在App的使用过程中，事件会在App中流转来寻找适合的响应者来针对事件响应。在本篇文章中。我们来了解下事件、响应者及响应链。 

## 响应者

首先我们来看下什么是响应者，响应者顾名思义是针对事件作出处理的对象。在 `iOS` 中响应者对象就是 `UIResponder`的实例。在 `UIKit` 框架中我们可以看到很多重要的对象都继承自`UIResponder`，包括：

- UIApplication对象 
- UIViewController
-  所有的`UIView`对象 (包括 `UIWindow `)。

![ UIResponder 继承图示](https://upload-images.jianshu.io/upload_images/1178839-138c78d02cce0317.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 事件
`App`收到一个事件后， 系统会自动派发事件到合适的响应者来处理。在`iOS`中`UIEvent`代表一个`UIKit`事件,目前`UIEvent`包含以下类型事件：

```
1. 触摸事件 touch events，最常见也是最常处理事件。
2. 运动事件 motion events，由`UIKit`触发的，与Core Motion框架息息相关。
3. 远程控制事件 remote-control events，远程控制事件允许响应者对象从外部附件或耳机接收命令，以便它可以管理音频和视频。
4. 按压事件 press events，常出现在 `game controller`, `AppleTV 遥控器, 或者其他具有物理按钮的设备交互时。

```

`ps:以下内容皆以触摸事件 touch events为例。`

在iOS中一个`touch`事件可能包含一个或者多个`UITouch`对象，如用户双指触碰屏幕，此时产生的一个触碰事件中有两个`UITouch`对象，每个`UITouch`对象对应一个手指。

而在一个触摸事件产生之后，系统会将事件传递给合适的`UIResponder`对象并调用相应的处理方法。在`UIResponder`中默认实现了响应的事件理方法，如处理`touch events`：

```
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesEnded:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesCancelled:(NSSet *)touches withEvent:(UIEvent *)event; // 外部打断，如电话的呼入。
```

![touches方法调用的时间点](https://upload-images.jianshu.io/upload_images/1178839-edbff6e2ecb2bcdd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# 如何寻找到合适的View处理事件

`UIKit`会用`Touch`事件的位置与视图层次结构中视图对象的`bounds`进行比较。遍历视图层次结构的`hitTest:withEvent:`方法，查找包含触摸事件的最深子视图，该子视图成为触摸事件的第一响应者。

大概步骤：

1. 判断`Touch`事件的`location`是否在自己`bounds`内。如果是进行下一步，如果不是则忽略
2. 遍历自身的子视图，`Touch`事件的`location`是否在子视图`bounds`。
3. 重复第一、第二步，直到没有子视图。则自己就是


UIView不能接收触摸事件的四种情况：

- 不允许交互：userInteractionEnabled = NO
- 隐藏：如果把父控件隐藏，那么子控件也会隐藏，隐藏的控件不能接受事件
- 透明度：如果设置一个控件的透明度<0.01，会直接影响子控件的透明度。- 
- alpha：0.0~0.01为透明。
- 父控件的 clipsToBounds = NO,子控件超出父控件的`bounds`。超出的控件不能接受事件

# 响应链

响应者接收事件数据后，必须处理事件或将其转发到另一个响应者对象。未处理的事件从响应者传递到下一个响应者，这个传递过程称之为响应链。

在`UIKit`的传递过程中，下一个响应者由`nextResponder`决定，许多`UIKit`类已经重写此属性并返回特定的对象，如下：

- UIView对象。
 - 如果视图是视图控制器的根视图，则下一个响应者是视图控制器；
 - 否则，下一个响应者是视图的父视图。
- UIViewController 对象。
 - 如果`UIViewController`的`view`是`window`的根视图（`rootViewController`），则下一个响应者是`window `对象。
 - 如果视图控制器是由另一个视图控制器`presented`的，则下一个响应者是`presentingViewController`。
 - `UIWindow`对象的下一个响应者是`UIApplication`对象。
 - `UIApplication`对象的下一响应者是 `AppDelegate`,当且`AppDelegate`是`UIResponder `对象，且不能是`view`、`view controller `和 `app`本身。

![响应链图示](https://upload-images.jianshu.io/upload_images/1178839-de689e025c9e7a99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
