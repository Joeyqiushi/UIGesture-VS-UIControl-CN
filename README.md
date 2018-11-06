# UIGesture和UIControl的前世今缘

最近发现很多同学都搞不清楚 UIGesture 和 UIControl 的正确使用姿势。即便是做了多年 iOS 开发的 senior engineer 也搞不清楚这整个脉络。于是我决定写一篇文章阐述一下这其中的奥妙。

一般来说，我们使用 UIGesture 和 UIControl 的场景大多比较简单。以 UIGesture 为例，
```
UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc] initWithTarget:self 
                                                                      action:@selector(tapGestureRecognized:)];
[view addGestureRecognizer:tap];
```
这可能是我们最常见的代码之一。大部分情况下它工作良好。但当其出现问题无法识别时，你是否会手足无措呢？

不同层级间的 UIGesture 是如何配合工作的？
当视图层级中嵌入着多个 UIGesture 和 UIControl 时，会不会互相影响？
在 UIButton 的父视图中添加 UIGesture ，可以被识别吗？
如果我在同一个视图上添加了多个 gesture ，哪一个会最终被识别呢？
为什么有时 UIButton 点击时的 highlight 状态会有延迟？

如果你对这些问题仍抱有疑问，本文希望给你一个答案。

## Steps for Gesture Recognition

首先，我们大体了解一下整个手势的识别流程。

 1. 在视图结构中找到响应链
 2. 手势之间建立依赖关系
 3. 决策哪些手势最终响应
 
接下来我们深入每个步骤去看看苹果是如何设计的。

## Response Chain

手势识别的第一步就是确定响应链。那么，什么是响应链呢？响应链是指，从你点击到的 view ，一直找其 superview ，直到根视图（通常是 UIWindow ），这个视图链我们称为响应链。举个例子:

![](https://coding.net/u/joeyxu/p/Resources/git/raw/master/ResponseChain.png)

红色的这个视图链，就是我们希望找到的响应链。之所以我们称其为响应链，是因为触摸事件将最终在这个视图链中得到处理。

Hit Test 是 Apple 设计的用于寻找响应链的工具。其实现如下：

```
// UIView.m
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    if (!self.isUserInteractionEnabled || self.isHidden || self.alpha < 0.01) {
        return nil;
    }
    if ([self pointInside:point withEvent:event]) {
        for (UIView *subview in [self.subviews reverseObjectEnumerator]) {
            CGPoint convertedPoint = [self convertPoint:point toView:subview];
            UIView *hitView = [subview hitTest:convertedPoint withEvent:event];
            if (hitView) {
                return hitView;
            }
        }
        return self;
    }
    return nil;
}
```

`[hitTest:withEvent:]` 和 `[pointInside:withEvent:]` 是最关键的两个方法。一开始，系统会调用根视图的 `[hitTest:withEvent:]`方法。根视图首先判断这个 touch point 是否在自己的区域内的。如果在，就继续倒叙遍历子视图看看这个 touch point 是否也落在了某个子视图内，如果是，则返回这个子视图的 `[hitTest:withEvent:]` 结果。这样 `[hitTest:withEvent:]` 就会对当前的视图结构进行一个深度遍历，直到找到最深的一个含有这个 touch point 的子视图。为了方便，这个最终找到的子视图在本文中被称为 TouchedView 。

苹果文档上有一点值得注意，那就是系统认为你点击到的视图可能和你看到的不一样。这种情况出现在子视图超出父视图的区域，而 clipsToBounds 为 false 时：
> Points that lie outside the receiver’s bounds are never reported as hits, even if they actually lie within one of the receiver’s subviews. This can occur if the current view’s clipsToBounds property is set to false and the affected subview extends beyond the view’s bounds.

## Gesture Dependancy

在响应链确认过后，系统开始建立手势之间的依赖关系。系统提供了两种设置依赖关系的方式，一种是初始化指定，一种是懒加载方式指定。我们先来看看懒加载的方式，那就是 `UIGestureRecognizerDelegate` 。

#### Build Dependancy Lazily - UIGestureRecognizerDelegate

在 Hit Test 确认响应链后，响应链中所有 UIGesture 的 `UIGestureRecognizerDelegate` 将会被调用。其调用步骤如下：

1. `[gestureRecognizer:shouldReceiveTouch:]` 被每个 gesture 调用一次，参数为 gesture 本身。如果调用返回 false ，那么该 gesture 将不会接收 touch 事件，也就没有了进一步识别手势的机会。如果返回 true , 那么系统会继续接下来第二步的调用。如果没有实现该方法，系统默认认为是返回 true 的。
2. `[gestureRecognizer:shouldRequireFailureOfGestureRecognizer:]` 和 `[gestureRecognizer:shouldBeRequiredToFailByGestureRecognizer:]` 将被每个 gesture 调用多次。对于每一个 gesture ，系统都允许其指定在另外任意一个 gesture 不响应时，它才响应。所以如果说响应链上有 N 个 gesture ，那么该方法将被每个 gesture 调用 N-1 次。如果不实现该方法，系统默认返回 false ，即没有失败依赖。另外，针对两个手势，只要任意一方返回 true ，失败依赖就会生效。
3. `[gestureRecognizerShouldBegin:]` 被每个 gesture 调用一次。但在这之前， TouchedView 的 `[gestureRecognizerShouldBegin:]` 将先被调用，并且会调用 N 次，每次调用所带的参数就是这个响应链上的 gesture 。这意味着当一个手势被识别时，系统首先会询问 TouchedView 是否允许响应链上的这些被识别的手势响应，如果 TouchedView 允许，再继续调用被允许的 gesture 的 `[gestureRecognizerShouldBegin:]` 方法，如果其也返回 true ，这个手势才会真正被允许响应。这里注意到， `[gestureRecognizerShouldBegin:]` 不仅是 `UIGestureRecognizerDelegate` 中的一个待实现方法，也在 UIView 中有着默认实现。这让 `[gestureRecognizerShouldBegin:]` 比较特殊，因为这个方法签名在两个地方均有使用，容易混淆。在 UIView 的默认实现中， `[gestureRecognizerShouldBegin:]` 返回 true 。但 UIView 的子类对其有不同的实现。另外，如果作为 `UIGestureRecognizerDelegate` 的 `[gestureRecognizerShouldBegin:]` 没有实现，那么系统默认认为返回的 true 。
4. `[gestureRecognizer:shouldRecognizeSimultaneouslyWithGestureRecognizer:]` 对所有被识别的 gesture ，其两两之间会被调用一次该方法。只要在任何一方的回调中返回 true ，双方就可以同时响应，反之。

经过以上4个步骤，我们通过 *UIGestureRecognizerDelegate* 建立了手势之间的依赖关系。

这里我分享一些 *UIGestureRecognizerDelegate* 的使用案例：
1. 如果我们希望 gesture 响应时，点击的 view 就是该 gesture 所属的 view 。即我们不希望我们在 view 上添加了一个 tap gesture 时，点击它的子视图这个 tap gesture 也会响应。这时我们可以利用 `[gestureRecognizer:shouldReceiveTouch:]` 这样实现：

```
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldReceiveTouch:(UITouch *)touch {
    if (touch.view == self.gestureRecognizer.view) {
        return YES;
    }
    return NO;
}
```

2. 如果我们想在视图链上添加了一个 tap gesture ，但我们希望新增的 tap gesture 不会影响到视图链上的 double tap gesture 。无论这个 double tap gesture 是现在已经存在，还是将来可能会有，我们都可以利用 
`[gestureRecognizer:shouldRequireFailureOfGestureRecognizer:]` 这样实现：

```
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRequireFailureOfGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer {
    // Double tap is prior.（otherGestureRecognizer includes all the legal gestures that might be recognized within the response chain）
    if ([otherGestureRecognizer isKindOfClass:[UITapGestureRecognizer class]]
        && ((UITapGestureRecognizer *)otherGestureRecognizer).numberOfTapsRequired == 2) {
        return YES;
    }
    return NO;
}
```

#### Build Dependancy Initially

除了懒加载之外，我们也可以通过一些 API 在初始化时就指定其依赖关系。比较典型的例子就是 *UIGestureRecognizer* 的 `[requireGestureRecognizerToFail:]` 方法。我们可以用其代替 `[gestureRecognizer:shouldRequireFailureOfGestureRecognizer:]` ：

```
[singleTap requireGestureRecognizerToFail:doubleTap];
```

和 *UIGestureRecognizerDelegate* 相比，初始化加载会更显笨拙。因为你必须有两个 gesture 的指针用来作为调用参数，而有时这两个 gesture 可能在代码和视图结构上距离很远，十分不便。并且规则的指定只限于已有的 gesture 对象，没法做一些统一的手势处理，例如上面提到的双击优先于单击响应。

## Gesture Recognized

在确认可响应的手势后，系统会倒叙遍历寻找最终得到响应的手势。在同一个视图中，后添加的手势优先响应。

![](https://raw.githubusercontent.com/Joeyqiushi/UIGesture-VS-UIControl-CN/master/Resource/DecideGesture.png)

如上图所示，假设我们在 ViewA 上添加了手势 gesture0 ， ViewB 上依次添加了手势 gesture1 和 gesture2 ， ViewC 上添加了 gesture3 。现在我们知道了响应链，也确认可响应的手势有 gesture0 ， gesture1 ， gesture2 ， gesture3 ，那么最终响应的手势就是 gesture3 。如果确认可响应的手势只有 gesture0 ， gesture1 ， gesture2 ，没有 gesture3 ，那么 gesture2 会得到响应。若没有 gesture2 ，那么 gesture1 得到响应。这样依次往前找到最终响应的手势。另外，在找到最终响应的手势后，若有任意一个 `[gestureRecognizer:shouldRecognizeSimultaneouslyWithGestureRecognizer:]` 回调中指定另外的手势可同时响应，并且另外的这些手势也属于可响应手势，那么这些手势会同时响应。

## UIControl

##### delayTouchesBegan

不知有没有读者遇到过 UIButton 的高亮状态延迟的情况？就是点击下去了，按钮过一会才会高亮。当你在 UIScrollView 上添加一个 UIButton 时，你就会发现这个现象。当然这个高亮延迟的现象还会出现在许多其他的场景中。这其中的奥妙，就在于 `delaysTouchesBegan` 。 UIScrollView 中有一个 gesture 叫做 `UIScrollViewDelayedTouchesBeganGestureRecognizer` ，这个 gesture 的 `delaysTouchesBegan` 属性为 true 。如果我们将其改为 fasle ，你会发现 UIButton 的高亮状态将不再延迟。我们在控制台打印一下 UIScrollView 的手势：

```
// one of the gestures on UIScrollView
<UIScrollViewDelayedTouchesBeganGestureRecognizer: 0x6000038c0100; state = Possible; delaysTouchesBegan = YES; view = <UIScrollView 0x7fde7580b600>; target= <(action=delayed:, target=<UIScrollView 0x7fde7580b600>)>>,
```

正如苹果文档中所说：
> When the value of this property is NO (the default), views analyze touch events in UITouchPhaseBegan and UITouchPhaseMoved in parallel with the receiver. When the value of the property is YES, **the window suspends delivery of touch objects in the UITouchPhaseBegan phase to the view**. If the gesture recognizer subsequently recognizes its gesture, these touch objects are discarded. If the gesture recognizer, however, does not recognize its gesture, the window delivers these objects to the view in a `touchesBegan:withEvent:` message (and possibly a follow-up touchesMoved:withEvent: message to inform it of the touches’ current locations). Set this property to YES to prevent views from processing any touches in the UITouchPhaseBegan phase that may be recognized as part of this gesture.

事实上 UIGesture 和 UIControl 的触摸响应机制是完全独立的两套。设置 `delaysTouchesBegan` 为 true 只会挂起 UIControl 的触摸响应事件，即 `[touchesBegan:withEvent:]` 被暂时挂起不被调用。但响应链中 UIGesture 的触摸事件的处理不会受到影响。若设置 `delaysTouchesBegan` 为 false ，那么 `[touchesBegan:withEvent:]` 会在 UIGesture 触摸识别完成后，在 `[gestureRecognizerShouldBegin:]` 之前被调用。所以当 `delaysTouchesBegan` 为 false 时， UIButton 在 `[touchesBegan:withEvent:]` 阶段就变成了高亮状态，尽管 UIGesture 最终被识别时 cancel 掉了所有的 UIControl 的触摸事件，也就是说最终 UIButton 的点击事件实际上是不会响应的。我想苹果设计 `delaysTouchesBegan` 这个属性应该是为了处理 UIGesture 和 UIControl 的兼容问题。所以尽管 `delaysTouchesBegan` 在某种程度上算是一种手势依赖，但我却把它放在了 UIControl 部分。因为它只会影响 UIControl 的 touch 事件，对 UIGesture 没有影响。

虽然 UIGesture 和 UIControl 有两套完全不同的 touch 处理机制，但其第一步确认响应链的过程是共享的。它们都通过 HitTest 来确认响应链。

除此之外，在触摸响应事件的处理中， UIGesture 比 UIControl 的优先级更高。为什么这么说呢？因为在响应链中只要有一个 UIGesture 拿到响应权，所有的 UIControl 的触摸事件都会被 cancel 掉，哪怕这个 UIControl 所处的视图层级比 UIGesture 的视图更高。简单点说， UIGesture 有权在其响应时中止 UIControl 的识别流程。

按上面所说，那么只要响应链中有 tap gesture ， UIButton 的点击事件就不会响应，取而代之的是 tap gesture 的响应。但事实好像并不是这样的。我们在一个含有 tap gesture 的 view 上添加一个 UIButton ，点击按钮时响应的是 UIButton 。为什么呢？事实上，这个 tap gesture 并没有获得响应权。问题出在 UIGestureRecognizerDelegate 的 `[gestureRecognizerShouldBegin:]` 阶段。在 `[gestureRecognizerShouldBegin:]` 阶段首先被调用的是被触摸视图的 `[gestureRecognizerShouldBegin:]` 方法，其参数是我们的 tap gesture 。而 UIButton 的 `[gestureRecognizerShouldBegin:]` 实现中，指定对非添加在自己身上的 tap gesture ，返回 false ，即不可响应。所以点击最终响应的是 UIButton ，其下面视图的 tap gesture 得不到响应。如果读者重写 UIButton 的 `[gestureRecognizerShouldBegin:]` 方法，让其返回 true ，会发现点击 UIButton 时， UIButton 没有响应，响应的却是其父视图的 tap gesture 。这也说明了 UIGesture 比 UIControl 的优先级更高。另外， UIButton 的 `[gestureRecognizerShouldBegin:]` 实现中没有对其他手势做限制，即返回的 true ，所以你在 UIButton 上双击、滑动时，这些手势都能得到其父视图的识别。

**总结一下，UIGesture 和 UIControl 的第一步响应链确认过程是一样的，都是 HitTest 。但他们的触摸识别机制是完全独立的两套。并且 UIGesture 的触摸事件响应流程的优先级高于 UIControl 。当 UIGesture 最终拿到响应权时，所有 UIControl 的触摸事件都会被立刻 cancel 掉，即中止识别。**

## In The End

希望本文对你在手势的使用和理解上有所帮助。
欢迎所有的问题和建议。
Have a good day! :)
