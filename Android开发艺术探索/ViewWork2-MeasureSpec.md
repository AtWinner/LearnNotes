MeasureSpec很大程度上决定了一个View的尺寸规格，之所以说是很大程度上因为这个过程还受父容器的影响，因为父容器影响View的MeasureSpec的创建过程，在测量过程中，系统会将View的LayoutParams根据父容器所施加的规则转换成对应的MeasureSpec，然后再根据这个measureSpec来测量出View的宽高。
# MeasureSpec
MeasureSpec代表一个32位的int值，高2位代表SpecMode，低30位代表SpecSize，SpecMode是指测量模式，而SpecSize是指在某种测量模式下的规格大小。
``` java

/** @hide */
@IntDef({UNSPECIFIED, EXACTLY, AT_MOST})
@Retention(RetentionPolicy.SOURCE)
public @interface MeasureSpecMode {}

/**
 * Measure specification mode: The parent has not imposed any constraint
 * on the child. It can be whatever size it wants.
 */
public static final int UNSPECIFIED = 0 << MODE_SHIFT;

/**
 * Measure specification mode: The parent has determined an exact size
 * for the child. The child is going to be given those bounds regardless
 * of how big it wants to be.
 */
public static final int EXACTLY     = 1 << MODE_SHIFT;

/**
 * Measure specification mode: The child can be as large as it wants up
 * to the specified size.
 */
public static final int AT_MOST     = 2 << MODE_SHIFT;

/**
 * Creates a measure specification based on the supplied size and mode.
 *
 * The mode must always be one of the following:
 * <ul>
 *  <li>{@link android.view.View.MeasureSpec#UNSPECIFIED}</li>
 *  <li>{@link android.view.View.MeasureSpec#EXACTLY}</li>
 *  <li>{@link android.view.View.MeasureSpec#AT_MOST}</li>
 * </ul>
 *
 * <p><strong>Note:</strong> On API level 17 and lower, makeMeasureSpec's
 * implementation was such that the order of arguments did not matter
 * and overflow in either value could impact the resulting MeasureSpec.
 * {@link android.widget.RelativeLayout} was affected by this bug.
 * Apps targeting API levels greater than 17 will get the fixed, more strict
 * behavior.</p>
 *
 * @param size the size of the measure specification
 * @param mode the mode of the measure specification
 * @return the measure specification based on size and mode
public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size, @MeasureSpecMode int mode) {
    if (sUseBrokenMakeMeasureSpec) {
        return size + mode;
    } else {
        return (size & ~MODE_MASK) | (mode & MODE_MASK);
    }
}
```
MeasureSpec通过将SpecMode和SpecSize打包成一个int值来避免过多的对象内存分配，为了方便操作，其提供了打包和解包的方法。SpecMode和SpecSize也是一个int值，一组SpecMode和SpecSize可以打包为一个MeasureSpec，而一个MeasureSpec可以通过解包的形式得出其原始的SpecMode和SpecSize，需要注意的是这里提到的MeasureSpec是指MeasureSpec所代表的int值，而不是MeasureSpec本身。

**SpecMode有三种分类：**
- **UNSPECIFIED：** 父容器不对View有任何限制，要多大给多大，这种情况一般用于系统内部，表示一种测量的状态。
- **EXACTLY：** 父容器已经检测出View所需要的精确大小，这个时候View的最终大小就是SpecSize所指定的值，它对应于LayoutParams中的match_parent和具体的数值这两种模式
- **AT_MOST：** 父容器指定了一个可用大小即SpecSize，View的大小不能大于这个值，具体是什么值要看不用View的具体实现，对应LayoutParams中的wrap_content。

# MeasureSpec和LayoutParams的对应关系
MeasureSpec不是唯一由LayoutParams决定的，LayoutParams需要和父容器一起才能决定View的MeasureSpec，从而进一步决定View的宽高。另外，++对于DecorView和普通View来说，MeasureSpec的转换过程略有不同。++ 对于DecorView，其MeasureSpec由窗口的尺寸和其自身的LayoutParams来共同决定；对于普通View，其MeasureSpec由父容器的MeasureSpec和自身的LayoutParams来共同决定。MeasureSpec确定之后，onMeasure就可以确定View的测量宽高。

MeasureSpec遵循的规则:
- LayoutParams.MATCH_PARENT：精确模式，大小就是窗口或父容器的大小；
- LayoutParams.WRAP_CONTENT：最大模式，大小不定，但是不能超过窗口大小；
- 固定大小（比如100dp）：精确模式，大小为LayoutParams中指定的大小。


ViewGroup的measureChildWithMargins方法
``` java
    /**
     * Ask one of the children of this view to measure itself, taking into
     * account both the MeasureSpec requirements for this view and its padding
     * and margins. The child must have MarginLayoutParams The heavy lifting is
     * done in getChildMeasureSpec.
     *
     * @param child The child to measure
     * @param parentWidthMeasureSpec The width requirements for this view
     * @param widthUsed Extra space that has been used up by the parent
     *        horizontally (possibly by other children of the parent)
     * @param parentHeightMeasureSpec The height requirements for this view
     * @param heightUsed Extra space that has been used up by the parent
     *        vertically (possibly by other children of the parent)
     */
    protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
上述方法会对子元素进行measure，在调用子元素的measure方法之前，会先通过getChildMeasureSpec方法来获取子元素的MeasureSpec。显然，子元素的MeasureSpec的创建和父容器的MeasureSpec和子元素本身的LayoutParams有关，此外还和View的margin和padding有关，具体情况如下：


``` java
    /**
     * Does the hard part of measureChildren: figuring out the MeasureSpec to
     * pass to a particular child. This method figures out the right MeasureSpec
     * for one dimension (height or width) of one child view.
     *
     * The goal is to combine information from our MeasureSpec with the
     * LayoutParams of the child to get the best possible results. For example,
     * if the this view knows its size (because its MeasureSpec has a mode of
     * EXACTLY), and the child has indicated in its LayoutParams that it wants
     * to be the same size as the parent, the parent should ask the child to
     * layout given an exact size.
     *
     * @param spec The requirements for this view
     * @param padding The padding of this view for the current dimension and
     *        margins, if applicable
     * @param childDimension How big the child wants to be in the current
     *        dimension
     * @return a MeasureSpec integer for the child
     */
    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //noinspection ResourceType
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }

```
它的主要作用是根据父容器的MeasureSpec同时结合View本身的LayoutParams来确定子元素的MeasureSpec，参数中的padding是指父容器中已经占用的空间大小，因此子元素的大小为父容器的尺寸减去padding

``` java
    int specSize = MeasureSpec.getSize(spec);
    int size = Math.max(0, specSize - padding);
```

# 综上所述
1. 当View采用固定宽高的时候，不管父容器的MeasureSpec是什么，View的MeasureSpec都是精确模式并且其大小遵循LayoutParams中的大小；
2. 当View的宽高是match_parent时，如果父容器的模式是精准模式，那么View也是精准模式，并且其大小是父容器的剩余空间；
3. 当View的宽高是match_parent时，如果父容器是最大模式，那么View也是最大模式并且大小不会超过父容器的剩余空间；
4. 当View的宽高是wrap_content时，不管父容器的模式是精准还是最大化，View的模式总是最大化并且大小不能超过父容器的剩余空间。
