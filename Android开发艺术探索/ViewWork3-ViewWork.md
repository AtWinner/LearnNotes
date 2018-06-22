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



