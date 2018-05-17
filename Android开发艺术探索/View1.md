# 写在前面
View是一个非常重要的概念，虽然说View不属于四大组件，但是它的作用堪比四大组件，甚至比Receiver和Provider的重要性都要大。在Android开发中，Activity承担了可视化的功能。

# View基础知识

## 什么是View
View是一种界面层的控件的一种抽象，它代表了一个控件。除了View还有ViewGroup，从名字来看，可以翻译成控件组。ViewGroup继承自View，这就意味着View本身就可以是单个控件也可以是由多个控件组成的控件，通过这种关系就形成了View树的结构，这和Web前端中的DOM树的概念是相似的。

## View的位置参数
View的位置主要由它的四个顶点来决定，分别对应于View的四个属性：top、left、right、bottom，其中top是左上角的纵坐标，left是左上角的横坐标，right是右下角的横坐标，bottom是右下角的纵坐标，我们很容易得出View的宽高和坐标的关系

```
width = right - left;
height = bottom - top;
```

从Android3.0开始，View增加了额外的几个参数：x、y、translationX和translationY，其中x和y代表View左上角的坐标，而translationX和translationY代表View左上角对于父容器的偏移量。这几个参数都是相对于父容器的坐标，并且translationX和translationY的默认值是0，和View的四个基本参数的位置一样，View也为它们提供了get/seet方法，换算关系如下：

```
x = left + translationX;
y = top + translationY;
```
需要注意的是，View在平移的过程中，top和left表示的是原始左上角的位置信息，其值并不会发生变化，此时发生改变的是x、y、translationX和translation。

## MotionEvent和TouchSlop
### MotionEvent
在手指解除屏幕后所产生的一系列事件中，典型的事件有：
- ACTION_DOWN：手指刚刚接触屏幕时
- ACTION_MOVE：手指在屏幕上移动时
- ACTION_UP：手指从屏幕上松开时

正常情况下，一次手指触摸屏幕的行为会出发一系列点击事件：
- 点击屏幕后离开松开，事件虚列为DOWN->UP；
- 点击屏幕滑动一会再松开，事件虚列为DOWN->MOVE->...->MOVE->UP

### TouchSlop
TouchSlop是系统所能识别出的被认为是滑动的最小距离，换句话说，当手指在屏幕上滑动时，如果两次滑动之间的距离小于这个常量，那么系统就不认为你是在进行滑动操作。这个常量的大小和设备有关，通常用如下方式获取：`ViewConfiguration.get(getContext()).getScaledTouchSlop()`。其作用就是使用这个常量来过滤掉滑动距离小于这个常量的滑动，这样可以获得更好的用户体验。可以在源码中找到这个常量的定义，在`frameworks/base/core/res/res/values/config.xml`。

```
<!-- Base "touch slop" value used by ViewConfiguration as a
    movement threshold where scrolling should begin. -->
<dimen name="config_viewConfigurationTouchSlop">8dp</dimen>
```

## VelocityTracker、GestureDetector和Scroller
### VelocityTracker
速度追踪，用于追踪手指在滑动过程中的速度，包括水平和竖直方向的速度。使用过程：
首先，在View和onTouchEvent方法中追踪当前单击事件的速度：

```
VelocityTracker velocityTracker = VelocityTracker.obtain();
velocityTracker.addMovement(event);
```
然后，当我们想知道当前的滑动速度时，可以采用如下方式来获得当前的速度：

```
velocityTracker.computeCurrentVelocity(1000);
int xVelocity = (int) velocityTracker.getXVelocity();
int yVelocity = (int) velocityTracker.getYVelocity();
```

需要注意：第一点，获取速度之前必须先计算速度，即getXVelocity和getYVelocity这两个方法的前面必须要调用computeCurrentVelocity方法；
第二点，这里的速度是指一段时间内手指所滑动过的像素。
速度的计算公式：`进度=（终点位置-起点位置）÷时间段`

## GestureDetector
手势检测，用于辅助检测用户的点击、滑动、长按、双击等行为。使用GestureDetector：

首先，需要创建一个GestureDetector对象并实现OnGestureListener接口，根据需要我们还可以实现OnDoubleTapListener从而能监听双击的行为：
```
GestureDetector mGestureDetector = new GestureDetector(this);
//解决长按屏幕后无法拖动的现象
mGestureDetector.setIsLongpressEnabled(false);
```
接着，接管目标View的OnTouchEvent方法，在待监听View的onTouchEvent方法中添加如下实现：

```
boolean consume = mGestureDetector.onTouchEvent(event);
return conusme;
```
然后，就可以有选择的实现OnGestureListener和OnDoubleTapListener中的方法了，方法介绍

<html>
<table width="1000">
  <tr>
    <th>方法名</th>
    <th>描述</th>
  </tr>
  <tr>
    <td>OnGestureListener<br/>#onDown</td>
    <td>手指轻轻触摸屏幕的一瞬间，由1个ACTION_DOWN</td>
  </tr>
  <tr>
    <td>OnGestureListener<br/>#onShowPress</td>
    <td>手指轻轻触摸屏幕，尚未松开或拖动，由1个ACTION_DOWN出发<br/>
    它强调的是没有松开或者拖动的状态</td>
  </tr>
  <tr>
    <td>OnGestureListener<br/>#onSingleTapUp</td>
    <td>手指松开，伴随着1个MotionEvent ACTION_UP而触发，这是单击行为</td>
  </tr>
  <tr>
    <td>OnGestureListener<br/>#onScroll</td>
    <td>手指按下屏幕并拖动，由1个ACTION_DOWN，多个ACTION_MOVE触发，这是拖动行为</td>
  </tr>
  <tr>
    <td>OnGestureListener<br/>#onLongPress</td>
    <td>用户长按屏幕不放</td>
  </tr>
  <tr>
    <td>OnGestureListener<br/>#onFling</td>
    <td>用户按下屏幕，快速的滑动后松开，由1个ACTION_DOWN、多个ACTION_MOVE和1个ACTION_UP触发，这是快速滑动行为</td>
  </tr>
  <tr>
    <td>OnDoubleTapListener<br/>#onDoubleTap</td>
    <td>双击，由两次连续的单击组成，它不可能和onSingleTapConfirmed共存</td>
  </tr>
  <tr>
    <td>OnDoubleTapListener<br/>#onSingleTapConfirmed</td>
    <td>严格的单击行为。注意它和onSingleTapUp的区别，如果触发了onSingleTapConfirmed，那么后面就不可能紧跟着另一个单击行为，即这只可能是单击，而不可能是双击中的一次单击</td>
  </tr>
  <tr>
    <td>OnDoubleTapListener<br/>#onDoubleTapEvent</td>
    <td>表示发生了双击行为，在双击的期间，ACTION_DOWN、ACTION_MOVE和ACTION_UP都会触发此回调</td>
  </tr>
</table>
</html>

### Scroller
弹性滑动对象，用于实现View的弹性滑动。我们知道，当使用View的scrollTo/scrollBy方法来进行滑动时，其过程是瞬间完成的，这是没有过渡效果的滑动用户体验不好，这个时候就可以使用Scroller来实现有过渡效果的滑动，其过程不是瞬间完成的，而是在一定时间内完成的。Scroller本身无法让View弹性滑动，它需要和View的computeScroll方法配合使用才能共同完成这个功能。

```
Scroller mScroller = new Scroller(mContext);

//缓慢滑动到指定位置
private void smoothScrollerTo(int destX, int destY) {
    int scrollX = getScrollX();
    int delta = destX - scrollX;
    //1000ms内滑向destX，效果就是慢慢滑动
    mScroller.startScroll(scrollX, 0, delta, 0, 1000);
    invalidate();
}

@Override
public void computeScroll(){
    if(mScroller.computeScrollOffset()) {
        scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
        postInvalidate();
    }
}
```






