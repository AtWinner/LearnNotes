# 常见的滑动冲突场景
1. 外部滑动方向和内部滑动方向不一致：比如ViewPager和Fragment的嵌套使用时，内部使用了一个ListView，本来这种情况下是会产生滑动冲突的，但是ViewPager内部处理了这种滑动冲突，因此采用ViewPager时我们无需关注这种冲突，但是如果我们不是用ViewPager而是ScrollView等，那就必须手动处理滑动冲突了。否则造成的后果就是内外两层只能有一层能够滑动，这是因为两者之间的滑动事件有冲突。
1. 外部滑动方向和内部滑动方向一致：当内外两层都在同一个方向可以滑动的时候，显然存在逻辑问题，因为手指开始滑动的时候，系统无法知道用户到底想让哪一层滑动
1. 上面两种情况的嵌套

从本质上讲，这三种滑动冲突场景的复杂度是相同的，因为它们的区别仅仅是滑动策略的不同，至于解决滑动冲突的办法，它们几个也是通用的

# 滑动冲突的解决方式
## 外部拦截法
点击事件都先经过父容器的拦截处理，如果父容器需要此事件就拦截，如果不需要此事件就不拦截，这样就解决了滑动冲突的问题，此方法比较符合事件的分发机制。外部拦截法需要重写父容器的onInterceptTouchEvent方法，再内部做相应的拦截即可。

``` java
public boolean onInterceptTouchEvent(MotionEvent event) {
    boolean intercepted = false;
    int x = (int) event.getX();
    int y = (int) event.getY();
    switch(event.getAction()) {
        case MotionEvent.ACTION_DOWN:{
            intercepted = false;
        }
        break;
        case MotionEvent.ACTION_MOVE: {
            if(父容器需要当前事件){
                intercepted = true;
            } else {
                intercepted = false;
            }
        }
        break;
        case MotionEvent.ACTION_UP:{
            intercepted = false;
        }
        break;
        default:
            break;
    }
    mLastXIntercept = x;
    mLayoutYIntercept = y;
    return intercepted;
}
```
上述代码是外部拦截法的典型逻辑，针对不同的华东冲突，只需要修改父容器需要当前点击事件这个条件即可，其他均不需要修改，也不能修改。这里对上述代码再描述一下，再onInterceptTouchEvent方法中，首先是ACTION_DOWN这个事件，父容器必须返回false，即不拦截ACTION_DOWN事件，这是因为一旦父容器拦截了ACTION_DOWN，那么后续的ACTION_MOVE和ACTION_UP事件都会直接交由父容器处理，这个时候之间没法再传递给子元素了；其次是ACTION_MOVE事件，这个事件可以根据需要来决定是否阻拦，如果父容器需要拦截就返回true，否则返回false；最后是ACTION_UP事件，这里必须要返回false，因为ACTION_UP事件本身没有太多意义。

## 内部拦截法
是指父容器不拦截任何事件，所有的事件都传递给子元素，如果子元素需要此事件就直接消耗掉，否则就交由父容器进行处理，这种方法和Android中的事件分发机制不一致，需要配合requestDisallowInterceptTouchEvent方法才能正常工作，使用起来比较外部拦截法比较复杂，我们需要首先重写子元素的dispatchTouchEvent方法：

```
public boolean dispatchTouchEvent(MotionEvent event) {
    int x = (int) event.getX();
    int y = (int) event.getY();
    
    switch(event.getAction()) {
        case MotionEvent.ACTION_DOWN:{
            parent.requestDisallowInterceptTouchEvent(true);
        }
        break;
        case MotionEvent.ACTION_MOVE:{
            int deltaX = x - mLastX;
            int deltaY = y - mLastY;
            if(父容器需要此类点击事件) {
                parent.requestDisallowInterceptTouchEvent(false);
            }
        }
        break;
        case MotionEvent.ACTION_UP:
        default:{
        }
        break;
        mLastX = x;
        mLastY = y;
        return super.dispatchTouchEvent(event);
    }
}
```
除了子元素需要做处理之外，父元素也要考虑默认拦截ACTION_DOWN以外的其他事件，这样当子元素调用parent.requestDisallowInterceptTouchEvent(false)方法时，父元素才会继续拦截所需的事件。因为ACTION_DOWN事件并不受FLAG_DISALLOW_INTERCEPT这个标记位的控制，所以一旦父容器拦截ACTION_DOWN事件，那么所有的事件都无法传递到子元素中去，这样内部拦截就无法起作用了。

