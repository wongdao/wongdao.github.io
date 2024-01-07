---
layout: post
title:  "RecyclerView Item 钉子户问题解决记"
slug: 2020-08-14-recyclerview-item-mess-ani
date: 2020-08-14T14:13:33+08:00
categories: android
---

之前是个超低概率出现的问题，很让人头疼，但所幸发现了让问题必现的方法，让问题得以顺利解决。大部分问题都是这样：不怕难，就怕难复现。

<!--more-->

注：
1. `RecyclerView` 下面图方便缩写为 **RV**
2. `LayoutManager` 则是 **LM**
3. `Property Animation` 是 **PA**

再说问题本身，问题的表现是，在刷新 RV 的数据时（先 clear 再 add 若干 数据），偶尔出现上一次的 item 被钉在屏幕上的情况，并且一旦 item 被钉一次，那么它之后就再也不会离开这个 RV ———— 有点像钉子户，所以标题就叫 RV 的钉子户问题。

这个现象我第一反应是 item view 脱离了 RV 的管理，应该和 RV 的 LM 有关（LM 负责item view 的展示）。RV 虽然继承自 ViewGroup，但管理 item view 的方式却是另辟蹊径搞了一套，所有子 view 的 **生老病死** 都不直接走 ViewGroup 的 api，而是全全由 RV 负责，如果不小心绕过了 RV 的 api 而直接使用了 ViewGroup 的 api 就肯定会出问题。

上面提到了“生老病死”，钉子户问题看起来应该是“死”这个环节的管理出了岔子。RV这个类本身只是个容器，它只提供管理机制而不提供管理策略，因此嫌疑不大，而负责管理策略的 LM 倒是很有嫌疑。于是我便对 LM 开始了检查。好在 app 的 LM 是自己撸的，因此查起来也算是熟门熟路，但无论检查多少遍 LM，就是没有任何发现。

幸运的是，这段时间因为有新的需求而加入了新的界面，新的界面很容易复现问题，在这个基础上，我留意到出现问题时都有动画还没结束就移动焦点的操作，可能和这两点有关？我做了个实验，先把动画时间延长到5秒，然后刷界面并且马上移动焦点，问题能100%复现了。看来问题的根源就在动画和焦点的配合出了问题。马上查看 item view 的代码，发现里面会在焦点改变时去设置自己的 animator 回调，这个操作会覆盖 RV 自带的 DefaultItemAnimator 设置的回调，导致动画结束后 item view 无法被回收。

最后将 item view 里的这段 `onFocusChanged` 代码删除，用其他实现方式替代之前的功能，问题解决。

----

简单说下 RV 动画的大致实现方式，可以分为三部分：

1. recycler view
   是一个容器，用于展示和管理 item view 的生命周期；
2. layout manager
   负责展示/排版 item views
3. predictive animation
   又是另辟蹊径的一套动画框架，可分为三个阶段：
   a. pre layout
   b. post layout
   c. **trigger animations** and do any necessary **cleanup**

换个角度，`3.c` 其实是个时间轴，开始是 `trigger animation`，结束是 on animation end，end 的时候去做 cleanup。如果只有 trigger 没有 on end，那么就不会触发 cleanup 从而导致问题。

RV 默认使用的 `DefaultItemAnimator` 类来实现动画，它使用了 Property Animation 的回调来做 cleanup，本文的钉子户问题则是在 trigger 后、on end 前抹掉了 Property Animation 的回调，导致最后没有 处理本应该回收的 item view。

以 add 操作为例，它使用 PA 回调的代码如下：

```java
void animateAddImpl(final RecyclerView.ViewHolder holder) {
    final View view = holder.itemView;
    final ViewPropertyAnimator animation = view.animate();
    mAddAnimations.add(holder);
    animation.alpha(1).setDuration(getAddDuration())
            .setListener(new AnimatorListenerAdapter() {
                ...
                @Override
                public void onAnimationEnd(Animator animator) {
                    animation.setListener(null);
                    dispatchAddFinished(holder);
                    mAddAnimations.remove(holder);
                    dispatchFinishedWhenDone();
                }
            }).start();
```

`dispatchAddFinished` 会对 item view 做回收操作。钉子户问题里 `onAnimationEnd` 不会被触发导致了 item view 没有被回收。

至此，问题解决，同时也搞清楚了全部的来龙去脉。

------

PS：如果 Property Animation 提供的设置回调 api 是 add 而不是 set，那么也不会出现上面的问题。当然这个可能性不大。
