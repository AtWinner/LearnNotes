Android动画可以分为3种：View动画、帧动画、属性动画。其中帧动画也属于View动画的一种，只不过它和平移、旋转等常见的View动画在表现形式上略有不同。
- View动画通过对场景里的对象不断做图像变换（平移、缩放、旋转、透明度）从而产生动画效果，**是一种渐进式动画**，并且View动画支持自定义。
- 帧动画通过顺序播放一系列图像从而产生动画效果，可以简单理解为图片切换动画，如果图片过大会导致OOM。
- 属性动画通过动态改变对象的属性从而达到动画效果，属性动画为API11的新特性，在低版本无法直接使用属性动画，但是我们仍然可以通过兼容库来使用它。

# View动画
View动画的作用对象是View，它支持4种效果，分别是平移、缩放、旋转、透明度
## View动画的种类
View动画的4个种类对应了Animation的4个子类：TranslateAnimation、ScaleAnimation、RotateAnimation、AlphaAnimation，这四种动画可以通过XML来定义，也可以通过代码创建，但是推荐使用XML，因为这样更可读

名称 | 标签 | 子类 | 效果
---|---|---|---
平移 | <translate>| TranslateAnimation | 移动View
缩放 | <scale> | ScaleAnimation | 放大或者缩小View
旋转 | <rotate> | RotateAnimation | 旋转View
透明度 | <alpha> | AlphaAnimation | 改变View的透明度

View动画既可以是单个动画，也可以由一系列动画组成

<set>标签标示动画的集合，对应AnimationSet类，它可以包含若干个动画，并且内部也可以嵌套其他动画集合，属性含义：

#### android:interpolator:
标示动画集合所采用的插值器，插值器影响动画的速度，比如非匀速动画就需要通多插值器来控制动画的播放过程，这个插值器可以不指定，默认为@android:anim/accelerate_decelerate_interpolator，即加速减速插值器。

#### android:shareInterpolator:
表示集合中的动画是否和集合共享同一个插值器。如果集合不指定插值器，那么子动画就需要单独指定所需的插值器或者使用默认值。

<translate>标签标示平移动画，对应TranslateAnimation类，它可以使一个View在水平和竖直方向完成平移的动画效果：
- android:fromXDelta:表示x的起始值，比如0；
- android:toXDelta:表示x的结束值，比如100；
- android:fromYDelta:表示y的起始值；
- android:toYDelta:表示y的结束值。

<scale>标签表示缩放动画，对应ScaleAnimation，它可以使View具有方法或者缩小的动画效果：
- android:fromXScale:水平缩放的起始值，比如0.5；
- android:toXScale:水平缩放的结束值，比如1.2；
- android:fromYScale:竖直方向缩放的起始值；
- android:toYScale:竖直方向缩放的结束值；
- android:pivotX:缩放的轴点的x坐标，它会影响缩放的效果；
- android:pivotY:缩放的轴点的y坐标，它会影响缩放的效果。

在<scale>标签中提到了轴点的概念，这里举个例子，默认情况下轴点是View的钟心点，这个时候在水平方向进行缩放的话会导致View向左右两个方向同时进行缩放，但是如果把轴点设成View的右边界，那么View就只会向左边进行缩放，繁殖则向右边进行缩放。

<rotate>标签表示旋转动画，对于RotateAnimation，它可以使View具有旋转的动画效果：
- android:fromDegrees:旋转开始的角度，比如0；
- android:toDegrees:旋转结束的角度，比如180；
- android:pivotX:轴点的x坐标；
- android:pivotY:轴点的y坐标。

在旋转动画中也有轴点的概念，它会影响到旋转的具体效果，轴点扮演着旋转轴的角色，即View是围绕着轴点进行旋转的，默认情况下轴点在View的中心点。

<aplha>标签表示透明度动画，对应AlphaAnimation，它可以改变View的透明度，它的属性的含义如下：
- android:fromAlpha:表示透明度的起始值，比如0.1；
- android:toAlpha:表示透明度的结束值，比如1。

# View动画的特殊使用场景
## LayoutAnimation
LayoutAnimation作用于ViewGroup，为ViewGroup指定一个动画，这样当它的子元素出场时都会具有这种动画效果，这种效果常常被用在ListView上，我们时长会看到一种特殊的ListView，它的每个item都会以一定的动画的形式出现，它用的就是LayoutAnimation。LayoutAnimation也是一个View动画，为了给ViewGroup的子元素加上出场效果，遵循如下效果
### 定义LayoutAnimation

``` xml
<LayoutAnimation 
    android:delay="0.5"
    android:animationOrder="normal"
    android:animation="@anim/anim_item" />

```
属性含义：

#### android:delay:
表示子元素开始动画的时间延迟，比如子元素入场动画的时间周期为300ms，那么0.5表示每个子元素都要延迟150ms才能播放入场动画，总体来说，第一个子元素延迟150ms开始播放入场动画，第2个子元素延迟300ms开始播放土场动画，一次类推。

#### android:animationOrder:
表示子元素动画的顺讯，有三种选项：normal、reverse、random，其中
- normal表示顺序显示，即排在前面的子元素先开始播放入场动画；
- reverse表示逆向显示，即排在后面的子元素开始播放入场动画；
- random是随机播放入场动画

#### android:animation:
为子元素指定具体的入场动画。

### 为子元素指定入场动画

``` xml
<set
    android:duration="300"
    android:interpolator="@android:anim/accelerate_interpolator"
    android:shareInterpolator="true">
    <alpha
        android:fromAlpha="0.0"
        android:toAlpha="1.0" />
    <translate
        android:fromXDelta="500"
        android:toXDelta="0" />
    
</set>
```

### 为ViewGroup指定android:layoutAnimation
android:layoutAnimation="@anim/xxx"对于ListView来说，这样的ListView的item就具有了出场动画，这种方法适用于所有的ViewGroup

# 属性动画
## 理解估值器和插值器
- TimeInterpolator中文翻译为时间插值器，它的作用是根据时间流逝的百分比来计算出当前属性值改变的百分比，系统预置的有LinearInterpolator（线性插值器：匀速动画）、AccelerateDecelerateInterpolator（加速减速插值器：动画两头慢中间快）、DecelerateInterpolator（减速插值器：动画越来越慢）。
- TypeEvaluator的中文翻译为类型估值算法，也叫估值器，它的作用是根据当前属性改变的百分比来计算改变后的属性值，系统预置的有IntEvaluator（针对整形属性）、FloatEvaluator（针对浮点型属性）和ArgbEvaluator（针对Color属性）
- 插值器和估值器很重要，它们是实现非匀速动画的重要手段。

由于动画的默认刷新率为10ms/帧，所以动画将分5帧进行，我们来考虑第三帧（x=20，r=20ms），当时间t=20ms的时候，时间流逝的百分比是0.5（20/40=0.5），意味着现在时间过去了一半，那x应该改变多少呢？这个就由插值器和估值器来确定。

``` java
public class LinearInterpolator extends BaseInterpolator implements NativeInterpolatorFactory {

    public LinearInterpolator() {
    }

    public LinearInterpolator(Context context, AttributeSet attrs) {
    }

    public float getInterpolation(float input) {
        return input;
    }

    /** @hide */
    @Override
    public long createNativeInterpolator() {
        return NativeInterpolatorFactoryHelper.createLinearInterpolator();
    }
}
```
很显然，线性插值器的返回值和输入值一样，因此插值器返回的值是0.5，这意味着x的改变时0.5，这个时候插值器的工作就完成了。具体x编程什么值，需要估值器来确实，看下整形估值器算法的源码：

``` java
public class IntEvaluator implements TypeEvaluator<Integer> {

    /**
     * This function returns the result of linearly interpolating the start and end values, with
     * <code>fraction</code> representing the proportion between the start and end values. The
     * calculation is a simple parametric calculation: <code>result = x0 + t * (v1 - v0)</code>,
     * where <code>x0</code> is <code>startValue</code>, <code>x1</code> is <code>endValue</code>,
     * and <code>t</code> is <code>fraction</code>.
     *
     * @param fraction   The fraction from the starting to the ending values
     * @param startValue The start value; should be of type <code>int</code> or
     *                   <code>Integer</code>
     * @param endValue   The end value; should be of type <code>int</code> or <code>Integer</code>
     * @return A linear interpolation between the start and end values, given the
     *         <code>fraction</code> parameter.
     */
    public Integer evaluate(float fraction, Integer startValue, Integer endValue) {
        int startInt = startValue;
        return (int)(startInt + fraction * (endValue - startInt));
    }
}
```
evaluate的三个参数分别表示估值小数、开始值和结束值，对应于我们的例子就是0.5、0、40，整形估值给我们的返回结果应该是20，这就是（x=20，t=20ms）的由来

属性动画要求对象的该属性有set方法和get方法（可选）。估值器和插值器除了系统提供的外，我们还可以定义，实现方法也很简单，因为插值器和估值器都是一个接口，且内部都只有一个方法，我们只需要派生一个类实现接口就可以了，就可以实现千奇百怪的动画效果了。

### 对任意属性做动画
属性动画要求对象的该属性有set方法和get方法，属性动画根据外界传递的该属性的初始值和最终值，以动画的效果多次去调用set方法，每次传递给set方法的值都不一样，所传递的值越来越接近最终值。我们对object的属性abc做动画，如果想让动画生效，要同时满足两个条件：
- object必须要提供setAbc方法，如果动画的时候没有传递初始值，那么还要提供getAbc方法，因为系统要去取的abc属性的初始值，如果不满足，程序直接Crash
- object的setAbc对属性abc所做的改变必须能通过某种方式反映出来，比如会带来UI的改变之类，如果不满足，动画无效果但是不会Crash

官方给出了3种解决方案：
- 给你的对象加上get和set方法，如果你有权限的话
- 用一个类来包装原始对象，间接为其提供get和set方法
- 采用ValueAnimator，监听动画过程，自己实现属性的改变
