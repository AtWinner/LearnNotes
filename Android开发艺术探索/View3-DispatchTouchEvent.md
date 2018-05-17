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
5. 如果View不消耗除ACTION_DOWN意外的其他事件，那么这个点击事件会消失，此时父元素的onTouchEvent并不会被调用，并且当前View可以持续收到后续的事件，最终这些消失的点击事件会传递给Activity处理。
6. ViewGroup默认不拦截任何事件，源码中的ViewGroup的onInterceptTouchEvent方法默认返回false。