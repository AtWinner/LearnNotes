# 传递规则
点击事件分发过程由3个很重要的方法共同完成：dispatchTouchEvent、onInterceptTouchEvent和onTouchEvent。

**public boolean dispatchTouchEvent(MotionEvent ev)**：用来进行事件的分发。如果事件能够传递给当前View，那么此方法一定会被调用，返回值受当前View的onTouchEvent和下级View的dispatchTouchEvent方法影响，++表示是否消耗当前事件++。

**public boolean onInterceptTouchEvent(MotionEvent ev)**：
在上述方法内部调用，用来判断是否拦截某个时间，如果当前View拦截了某个事件，那么在同一个事件序列中，此方法不会被再次调用，返回结果表示是否拦截当前事件。

**public boolean onTouchEvent(MotionEvent ev)** ：在dispatchTouchEvent方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前View无法再次接收到事件。

下面一段伪代码描述了上述3个方法的区别

```
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean consume = false;
    if(onInterceptTouchEvent(ev)) {
        consume = onTouchEvent(ev);
    } else {
        consume = child.dispatchTouchEvent(ev);
    }
    return consume;
}
```
对于ViewGroup来说，它的dispatchTouchEvent会被调用，如果这个ViewGroup的onInterceptTouchEvent方法返回true就表示它要拦截当前事件，接着事件就会交给这个ViewGroup处理，它的onTouchEvent方法就会被调用；如果ViewGroup的onInterceptTouchEvent方法返回false就表示它不拦截当前事件，这时当前事件就会继续传递给它的子元素，接着子元素发dispatchTouchEvent方法就会被调用，如此反复直到事件被最终处理。

对于View，如果它设置了OnTouchListener，那么OnTouchListener中的onTouch方法会被回调。这时事件如何处理还要看onTouch的返回值，如果返回false，则当前View的onTouchEvent方法会被调用；如果返回true，则不会被调用。由此可见，给View设置的onTouchListener，其优先级比onTouchEvent要高。在onTouchEvent方法中，如果当前设置的有OnClickListener，那么它的onClick方法会被调用。可以看出，平时我们常用的OnClientListener，其优先级最低，处于事件传递的尾端。

一个点击事件产生之后，它的传递过程遵循如下顺序：Activity->Window->View

关于事件传递机制，这里给出了一些结论：
1. 同一个事件序列是指从手指触摸屏幕的那一刻起，到手指离开屏幕的那一刻结束，在这个过程中所产生的一系列事件，这个事件序列以down事件开始，中间含有数量不定的move事件，最终以up事件结束。
2. 正常情况下，一个事件序列只能被一个View拦截且消耗。因为一旦一个元素拦截了某些事件，那么同一个事件序列内的所有事件都会直接交给它处理，因此同一个事件序列中的事件不能分别由两个View处理，比如一个View本该自己处理的事件通过onTouchEvent强行传递给其他View处理。
3. 某个View一旦决定拦截，那么这一个事件序列就都只能由它处理，并且它的onInterceptTouchEvent不会再被调用。
4. 某个View一旦开始处理事件，如果它不消耗ACTION_DOWN事件，那么同一事件序列中的其他事件都不会再交给它来处理，并且事件将重新交给它父元素去处理，即父元素的onTouchEvent会被调用。意思就是事件一旦交给一个View处理，那么它就必须消耗掉，否则同一事件序列中剩下的事件都不会再交给它处理了。
5. 如果View不消耗除ACTION_DOWN以外的其他事件，那么这个点击事件会消失，此时父元素的onTouchEvent并不会被调用，并且当前View可以持续收到后续的事件，最终这些消失的点击事件会传递给Activity处理。
6. ViewGroup默认不拦截任何事件，源码中的ViewGroup的onInterceptTouchEvent方法默认返回false。
7. View没有onInterceptTouchEvent方法，一旦有点击事件传递给它，那么它的onTouchEvent方法就会被调用。
8. View的onTouchEvent默认都会消耗事件，除非它是不可点击的，==clickable和longClickable同时为false==。View的longClickable默认都为false，clickable要分情况，比如Button的clickable为true，TextView的clickable为false。
9. View的enable属性不影响onTouchEvent的默认返回值。
10. onClick会发生的前提是当前View是可点击的，并且它收到了down和up的事件。
11. 事件传递过程是由外向内的，即事件总是先传递给父元素，然后再由父元素分发给子View，通过requestDisallow

# Activity事件分发
当一个点击操作发生时，事件最先传递给当前的Activity，由Activity的dispatchTouchEvent来进行事件分发，具体的工作室由Activity内部的Window来完成的。Window会将事件传递给decor view，decor view一般就是当前界面的底层容器，即setContentView所设置的View父容器，通过Activity.getWindow.getDecorView()可以获得。

```
/**
* Called to process touch screen events.  You can override this to
* intercept all touch screen events before they are dispatched to the
* window.  Be sure to call this implementation for touch screen events
* that should be handled normally.
*
* @param ev The touch screen event.
*
* @return boolean Return true if this event was consumed.
*/
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```
Window#superDispatchTouchEvent方法是一个抽象方法，它只在android.policy,PhoneWindow中实现了：

```
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
    //直接传递给了decorView处理了
}
```

# 顶级View对点击事件的分发
如果顶级ViewGroup拦截事件即onInterceptTouchEvent返回true，则事件由ViewGroup处理，这时如果ViewGroup的MOnTouchListener，则onTouch会被调用，否则onTouchEvent会被调用。也就是说，`onTouch会屏蔽掉onTouchEvent`。在onTouchEvent中，如果设置了mOnClickListener，则onClick会被调用。如果顶级ViewGroup不拦截事件，则事件会传递给它所在的点击事件链上的子View，这时子View的dispatchTouchEvent会被调用。

![事件分发传递过程](http://oi9a3yd8k.bkt.clouddn.com/dispatchTouchEvent.png)

```
// Handle an initial down.
if (actionMasked == MotionEvent.ACTION_DOWN) {
    // Throw away all previous state when starting a new touch gesture.
    // The framework may have dropped the up or cancel event for the previous gesture
    // due to an app switch, ANR, or some other state change.
    cancelAndClearTouchTargets(ev);
    resetTouchState();
}
```
onInterceptTouchEvent不是每次事件都会被调用，如果我们想提前处理所有的点击事件，要选择dispatchTouchEvent方法，只有这个方法能确保每次都会调用，当然前提是事件能够传递到当前ViewGroup
