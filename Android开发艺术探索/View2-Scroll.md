通过三种方式可以实现View滑动：
1. 通过View本身提供的scrollTo/scrollBy方法来实现滑动；
1. 通过动画给View施加平移效果来实现滑动；
1. 通过改变View的LayoutParams是的View重新布局从而实现滑动。

# 使用scrollTo/scrollBy

```
    /**
     * Set the scrolled position of your view. This will cause a call to
     * {@link #onScrollChanged(int, int, int, int)} and the view will be
     * invalidated.
     * @param x the x position to scroll to
     * @param y the y position to scroll to
     */
    public void scrollTo(int x, int y) {
        if (mScrollX != x || mScrollY != y) {
            int oldX = mScrollX;
            int oldY = mScrollY;
            mScrollX = x;
            mScrollY = y;
            invalidateParentCaches();
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (!awakenScrollBars()) {
                postInvalidateOnAnimation();
            }
        }
    }

    /**
     * Move the scrolled position of your view. This will cause a call to
     * {@link #onScrollChanged(int, int, int, int)} and the view will be
     * invalidated.
     * @param x the amount of pixels to scroll by horizontally
     * @param y the amount of pixels to scroll by vertically
     */
    public void scrollBy(int x, int y) {
        scrollTo(mScrollX + x, mScrollY + y);
    }

```

scorllBy实际上也是调用了scrollTo方法，它实现了基于当前位置的相对滑动，而scrollTo则实现了基于所传递参数的绝对滑动。利用scrollTo和scrollBy来实现View的滑动，这不是一件困难的事情，但是我们要明白滑动过程中View内部的两个属性mScrollX和mScrollY的改变规则，这两个属性可以通过getScrollX和getScrollY方法分别得到。在滑动的过程中，mScrollX的值总是等于View左边缘和View内容左边缘在水平方向上的距离，而mScrollY的值总是等于View左边缘和View内容左边缘在垂直方向上的距离。View的边缘是指View的位置，由四个顶点组成，而View内容边缘是指View中的内容的边缘，scrollTo和scrollBy只能改变View内容的位置而不能改变View在布局中的位置。mScrollX和mScrollY的单位是像素。如果从左向右滑，那么mScrollX为负数，反之为正值；如果从上往下滑动，那么mScrollY为负数，反之为正值。


