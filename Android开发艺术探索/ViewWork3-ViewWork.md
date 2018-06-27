View的工作流程主要是指measure、layout、draw这三大流程，即测量、布局和绘制

# measure过程
measure的过程要分情况来看，如果只是一个原始的view，那么通过measure方法就完成了它的测量过程；

如果是一个ViewGroup，除了完成自己的测量过程之外，还会遍历调用所有子元素的measure方法，递归以上的流程。
## View的measure过程
View的measure过程是由measure方法来完成的，measure方法是一个final的方法，子类不能重写这个方法，在View的measure方法中回去调用View的onMeasure方法，因此只需要看看onMeasure的实现即可。

``` java
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
    
        /**
     * Utility to return a default size. Uses the supplied size if the
     * MeasureSpec imposed no constraints. Will get larger if allowed
     * by the MeasureSpec.
     *
     * @param size Default size for this view
     * @param measureSpec Constraints imposed by the parent
     * @return The size this view should be.
     */
    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }

```
对于AT_MOST和EXACTLY，其实，getDefaultSize返回的大小就是measureSpec中的SpecSize，而这个SpecSize就是View测量后的大小，这里多次提到了测量后的大小，是因为View的最终大小是在layout阶段确定的 ，所以这里必须要加以区分，但是几乎所有情况下的View的测量大小和最终大小是相同的。

对于UNSPECIFIED这种情况，一般用于系统内部的测量过程，在这种情况下View的大小为getDefaultSize的第一个参数size，即宽高分别为getSuggestedMinimumWidth和getSuggestedMinimumHeight这两个方法的返回值：

``` java
    protected int getSuggestedMinimumHeight() {
        return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());

    }
    protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
```

如果View没有设置背景，那么View的宽度为mMinWidth，而mMinWidth对应android:minWidth这个属性所指定的值，因此View的宽度即为minWidth所指定的值，如果这个属性没有指定，那么minWidth的默认值为0,；如果View设置了背景，则View的宽度为max(mMinWidth, mBackground.getMinimumWidth())，getMinimumWidth()逻辑如下：

``` java
    public int getMinimumWidth() {
        final int intrinsicWidth = getIntrinsicWidth();
        return intrinsicWidth > 0 ? intrinsicWidth : 0;
    }
```
可以看出getMinimumWidth返回的是Drawable的原始宽度，前提是这个Drawable有原始宽度，否则返回0。

从getDefaultSize方法的实现看来，View的宽高由specSize决定，所以我们可以得出以下结论：直接继承View的自定义控件需要重写onMeasure方法并设置warp_content时的自身大小，否则在布局中使用wrap_content就相当于使用match_parent，对于这种情况，我们只需要给View指定一个默认的内部宽高并在wrap_content时设置宽高即可。对于非wrap_content情形，我们沿用系统的测量值即可，至于这个默认的内部宽高如何指定，没有固定的根据，根据需要灵活指定即可，如果查看TextView和ImageView的源码就可以知道，针对wrap_content情形，它们的onMeasure方法均作了特殊处理。

## ViewGroup的measure过程
对于ViewGroup来说，除了完成自己的measure过程，还会遍历去调用子元素的measure方法，各个子元素再递归去执行这个过程。和View不同的是，ViewGroup是一个抽象类，因此它没有重写View的onMeasure方法，但是提供了一个叫measureChildren的方法：

``` java
    /**
     * Ask all of the children of this view to measure themselves, taking into
     * account both the MeasureSpec requirements for this view and its padding.
     * We skip children that are in the GONE state The heavy lifting is done in
     * getChildMeasureSpec.
     *
     * @param widthMeasureSpec The width requirements for this view
     * @param heightMeasureSpec The height requirements for this view
     */
    protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }
```
ViewGroup在measure时会对每个子元素进行measure，measureChild这个方法的实现也很好理解：

``` Java
    /**
     * Ask one of the children of this view to measure itself, taking into
     * account both the MeasureSpec requirements for this view and its padding.
     * The heavy lifting is done in getChildMeasureSpec.
     *
     * @param child The child to measure
     * @param parentWidthMeasureSpec The width requirements for this view
     * @param parentHeightMeasureSpec The height requirements for this view
     */
    protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }

```
measureChild的思想就是去除子元素的LayoutParams，然后再通过getChildMeasureSpec来创建子元素的MeasureSpec，接着将MeasureSpec直接传递给View的measure方法来进行测量。

ViewGroup没有定义其测量的具体过程，这是因为ViewGroup是一个抽象类，其测量过程的onMeasure方法需要各个子类具体实现，比如LinearLayout、RelativeLayout等。不同的ViewGroup子类会有不同的布局特性，这导致他们的测量细节各有不同，比如LinearLayout和RelativeLayout这两个布局特性完全不同。

``` java
    //LinearLayout的onMeasure
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        if (mOrientation == VERTICAL) {
            measureVertical(widthMeasureSpec, heightMeasureSpec);
        } else {
            measureHorizontal(widthMeasureSpec, heightMeasureSpec);
        }
    }
```

