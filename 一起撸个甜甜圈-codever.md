# 一起撸个甜甜圈吧

**本文首发掘金（没写完orz），现重写并补全。**

在Android中，说到图表，我们往往都会选择找库，比如MPAndroidChart。

然而更多的时候，我们往往只需要某一类型的图表，为了这个类型的图表而不得不把整个库（包含所有图表逻辑）导入进来，还是感觉有点重的。

俗话说得好，自己动手，丰衣足食。

于是就有了今天的甜甜圈。（为何叫甜甜圈。。。不觉得环形饼图好像个甜甜圈吗哈哈）

![](https://github.com/razerdp/AnimatedPieView/blob/master/art/pie_click_effect.gif)

---

### 起源
作为程序员，一个新的需求/控件的起源，很多时候都是来源于产品，所以，这次控件的诞生，其实很简单，来源于一张优化点设计图 **（只截取部分，其余部分不宜公开）**：
![](https://github.com/razerdp/Article/blob/master/pics/一起撸个甜甜圈/1.jpg)

咋一看，这张图so easy啦~ 3只paint，3个颜色，随便画画，搞定~

可是，不知道为啥，看着这个甜甜圈还挺漂亮的，不甘心就这么简单的画出来就完事。。。

于是心里有个魅惑的声音告诉我：“既然这个不急，只是优化点，那为何不优化彻底一点，做的比这货更漂亮呢”

被魅惑的我，立马熟悉的打开AS，新建Project，填上Name，弄个类继承View，然后。。。。

然后。。。他喵的然后怎么干啊！！！！

### 迷茫

相信很多人写自定义控件都会有这个疑惑。。。我建好类了~ 我继承好了View了~ 接下来，，，我不会做了TAT

其实不仅仅你们，就是我写过不少的控件，现在建好一个类，我也偶尔会有这个情况。。。（嗯，可能写的还是不够多）。

原因很简单：**所有的元素，都是心里所想，并没有做出一个具体的动画效果和预览，所以方向太多，一下子很迷茫。**

因为当我们想完成一个控件或者动画效果是，我们往往很快就能在心里确定好我们想要什么效果，然而当我们真的要实施的时候，就会发现似乎不知道该怎么把效果变成一行行的代码。

在迷茫的时候，我的做法是，把我的期望写下来：

* 我的甜甜圈会动
* 我的甜甜圈可配置参数要多（自由度要高）
* 我的甜甜圈点击的时候要有个效果，至于效果，大概是浮上来呈现出Z轴上移的样子，最好能加上阴影
* 我的甜甜圈要简单好上手。。。

当需求写了下来，我们就可以逐步击破。

针对上面的需求点，我们逐步确定我们的方案：

* 甜甜圈
    * 继承View，自己画
* 会动的甜甜圈
    * Animation走起，反正就是可以逐步计算进度并让我根据值来进行不同的绘制即可
* 可配置多
    * 因为参数比较多，因此归并到一个config类里面，采取Builder模式，形成链式配置，保持清爽的编码风格
* 点击效果
    * 甜甜圈可以通过大小变化来造成z轴上浮的伪效果，加上BlurMaskFilter或者ShadowLayer
* 简单上手
    * 暴露的api尽可能少，以及面向接口编程
    
### 数据

受限于篇幅，这里仅贴出核心代码，其余地方以思路讲解为主。

首先由简入繁，我们先尝试画出一个简单的甜甜圈：

```java
public class AnimatedPieView extends View {
    protected final String TAG = this.getClass().getSimpleName();

    private Paint paint1;
    private Paint paint2;
    private Paint paint3;

    RectF mDrawRectf = new RectF();


    //其他构造器忽略
    public AnimatedPieView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        initView(context, attrs);
    }


    private void initView(Context context, AttributeSet attrs) {
        paint1 = new Paint(Paint.ANTI_ALIAS_FLAG | Paint.DITHER_FLAG);
        paint1.setStyle(Paint.Style.STROKE);
        paint1.setStrokeWidth(80);
        paint1.setColor(Color.RED);

        paint2 = new Paint(paint1);
        paint2.setColor(Color.GREEN);

        paint3 = new Paint(paint1);
        paint3.setColor(Color.BLUE);

    }


    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        final float width = getWidth() - getPaddingLeft() - getPaddingRight();
        final float height = getHeight() - getPaddingTop() - getPaddingBottom();

        canvas.translate(width / 2, height / 2);
        //半径
        final float radius = (float) (Math.min(width, height) / 2 * 0.85);
        mDrawRectf.set(-radius, -radius, radius, radius);

        canvas.drawArc(mDrawRectf, 0, 120, false, paint1);
        canvas.drawArc(mDrawRectf, 120, 120, false, paint2);
        canvas.drawArc(mDrawRectf, 240, 120, false, paint3);


    }

}
```

so easy~依然是开头所说的，3只笔，3个角度，完事（极限一点，一支笔也可以完成【静态的情况下】）。

![](https://github.com/razerdp/Article/blob/master/pics/一起撸个甜甜圈/3.jpg)

然而，如果我们允许外部配置更多的甜甜圈的话，那岂不是要修改这个类？如果有一个方法能够提供给我动态增删就好了。

说做就做，我们看一下目前甜甜圈需要的元素：

* 颜色
* 描述
* 数据（计算比例用）

也就是说，如果我们跟用户约定一个方式，约定好由用户提供这些元素，我们仅负责渲染的话，那么我们就可以实现动态增删的需求了。

那么，该如何约定呢？

假如我们要求使用一个类，也就是用户必须传给我们这个类，那么对于用户来说，是一个麻烦的使用，因为我们想绘制在图标上的数据往往都是服务器请求回来的，如果想画出甜甜圈，还得做多一步转换，这无疑是增大了复杂度，也就不符合我们简单好用的期望了。

考虑到这个，我采取了接口的方式。接口约束要求返回以上三个元素，这样做的好处是不用破坏用户原来的数据，他甚至可以用原来的数据bean来实现甜甜圈的需求，另一个好处是我可以很方便的进行扩展。

事实上，这一点也得到了使用者的认可（ps，在国外本库被评为2018年初值得关注的25个库之一，因此国外issue提出的比较多，邮件联系的也是主要是外国友人）：

![](https://github.com/razerdp/Article/blob/master/pics/一起撸个甜甜圈/4.png)

根据以上条件，我们初步定义出一个接口：IPieInfo

```java
public interface IPieInfo {

    double getValue();

    @ColorInt
    int getColor();

    String getDesc();
}
```

在这之后，我们计算都可以依靠接口获取：

* 获取颜色决定本段甜甜圈的取色
* 获取数值来计算本段甜甜圈的
* 获取描述来绘制甜甜圈的描述

这样做可以不限制用户的数据类型，只需要实现我们的接口并对各个方法进行返回即可。

在绘制的时候，我们把用户传入的数据存到一个List中，在存入的时候，根据获取的value进行计算，得到其开始角度和结束角度，并把数据包装在一个类里面供控件内部使用。而这一切，对外部来说都是不透明的，外部使用仅仅关注的是配置，而不是计算。

### 点击

甜甜圈的点击是一个比较麻烦的点，主要原因如下：

* 甜甜圈支持起始角度设置，而对起始角度并没有做要求，也就是传入-3600°也是可以的
* 点击的时候需要精确判定点击的区域在哪个甜甜圈里
* 甜甜圈被点击后的动作，以及上一次点击的甜甜圈动画和本次点击的甜甜圈动画需要切换（一个还原一个上浮）

首先看看第一个问题，我们的甜甜圈虽然可以设置无限角度，但实际上其实归根结底可以归到0 ~ 360°之间，即便传入一个很大的值，其实也是一定倍数 * 360 + 偏移量而已，所以针对任意角度，我们需要将其收束到0 ~ 360°之间：

```java
public class DegreeUtil {

    public static float limitDegreeInTo360(double inputAngle) {
        float result;
        double tInputAngle = inputAngle - (int) inputAngle;//取小数
        result = (float) ((int) inputAngle % 360.0f + tInputAngle);
        return result < 0 ? 360.0f + result : result;
    }
}
```

在点击触发的时候，我们先判断点击的位置是否在甜甜圈（或者饼图）内，判断的方法也很简单，就是初中的技巧计算两点之间的直线距离。

我们获取触摸点的x,y，计算其到中心的距离，假如当前是甜甜圈模式（环形饼图），则需要甜甜圈内径≤距离≤甜甜圈外径则判定在甜甜圈内。

```java
        PieInfoWrapper pointToPieInfoWrapper(float x, float y) {
            final boolean isStrokeMode = mConfig.isStrokeMode();
            final float strokeWidth = mConfig.getStrokeWidth();
            //外圆半径
            final float exCircleRadius = isStrokeMode ? pieRadius + strokeWidth / 2 : pieRadius;
            //内圆半径
            final float innerCircleRadius = isStrokeMode ? pieRadius - strokeWidth / 2 : 0;
            //点击位置到圆心的直线距离(没开根)
            final double touchDistancePow = Math.pow(x - centerX, 2) + Math.pow(y - centerY, 2);
            //内圆半径<=直线距离<=外圆半径
            final boolean isTouchInRing = touchDistancePow >= expandClickRange + Math.pow(innerCircleRadius, 2)
                    && touchDistancePow <= expandClickRange + Math.pow(exCircleRadius, 2);
            if (!isTouchInRing) return null;
            return findWrapper(x, y);
        }
```

计算完距离后，我们需要计算角度，我们通过角度来获取我们当前点击的是哪一段的甜甜圈。

目前我们已知点击的xy，以及中心点，这时候我们可以用atan2方法反计算出角度
```java
    //得到角度
    double touchAngle = Math.toDegrees(Math.atan2(y - centerY, x - centerX));
```

这里计算出来的是-180° ~ 180°范围内的值，即以x正半轴为起始，逆时针（1、2象限）则是-180° ~ 0，顺时针（3、4象限）是0 ~ 180°。而我们的甜甜圈在开始的时候也说过，是无限角度的，当然，我们处理到0 ~ 360°，然而即便收束了起来，还是与我们计算出来的-180° ~ 180°对不上，因此我们对计算出来的角度需要做一下处理。

在这里我选择当点击的角度小于0的时候，加上360°。这里可能会有小伙伴问我为什么不是加上180°，而是加360°。

这个问题很简单，因为我们的甜甜圈在转换为0 ~ 360°之后，在1、2象限表现的是180° ~ 360°的范围，而我们点击的角度在1、2象限是-180° ~ 0，如果加上180，则是0 ~ 180°，还是无法满足我们的甜甜圈判断，因此在数值小于0的情况下，我们加上360°。

![](https://github.com/razerdp/Article/blob/master/pics/一起撸个甜甜圈/8.png)

接着我们根据角度寻找每一段甜甜圈里匹配的角度进行查询，直到找到为止。

```java
        PieInfoWrapper findWrapper(float x, float y) {
            //得到角度
            double touchAngle = Math.toDegrees(Math.atan2(y - centerY, x - centerX));
            if (touchAngle < 0) {
                touchAngle += 360.0f;
            }
            if (lastTouchWrapper != null && lastTouchWrapper.containsTouch((float) touchAngle)) {
                return lastTouchWrapper;
            }
            PLog.i("touch角度 = " + touchAngle);
            for (PieInfoWrapper wrapper : mDataWrappers) {
                if (wrapper.containsTouch((float) touchAngle)) {
                    lastTouchWrapper = wrapper;
                    return wrapper;
                }
            }
            return null;
        }
```

```java
    boolean containsTouch(float angle) {
        //所有点击的角度都需要收归到0~360的范围，兼容任意角度
        final float tAngle = DegreeUtil.limitDegreeInTo360(angle);
        float tStart = DegreeUtil.limitDegreeInTo360(fromAngle);
        float tEnd = DegreeUtil.limitDegreeInTo360(toAngle);
        PLog.d("containsTouch  >>  tStart： " + tStart + "   tEnd： " + tEnd + "   tAngle： " + tAngle);
        boolean result;
        if (tEnd < tStart) {
            if (tAngle > 180) {
                //已经过界
                result = tAngle >= tStart && (360 - tAngle) <= sweepAngle;
            } else {
                result = tAngle + 360 >= tStart && tAngle <= tEnd;
            }
        } else {
            result = tAngle >= tStart && tAngle <= tEnd;
        }
        if (result) {
            PLog.i("find touch point  >>  " + toString());
        }
        return result;
    }
```

在查找的时候我们还需要注意转换为0 ~ 360的情况中有一种特殊情况，就是某段甜甜圈跨越了0和360的界限。比如说图中的情况：
![](https://github.com/razerdp/Article/blob/master/pics/一起撸个甜甜圈/11.png)


其他的部分比如点击的动画实现等，则是由Animator计算并不断重绘。这里就不再详细说明。


### 文字

文字的绘制相对简单，我们需要确定的是文字的位置就可以了。

从效果图上我们知道，文字绘制有个引导线，而文字要么在引导线上，要么在引导线下，要么上下都有。

为了扩展，此处我粗暴的给出了四种文字属性：

* 文字都在引导线上
* 文字都在引导线下
* 文字在1、2象限在引导线上，在3、4象限处于引导线下
* 文字与引导线对齐

![](https://github.com/razerdp/Article/blob/master/pics/一起撸个甜甜圈/16.jpg)

计算文字的位置首当其冲我们得确认文字所处象限，然而我们仅仅知道的条件只有这段甜甜圈的起始、结束角度、甜甜圈半径，因此我们需要把角度换算为距离。

那我们需要怎么做呢？实际上这也是三角函数的简单运用，根据效果图，文字指引线的起始点是在甜甜圈的中间，因此我们可以根据甜甜圈的中心点到某段甜甜圈的中间连线作为三角形的斜边，根据三角函数sin/cos求出x,y的坐标即可。

```java
    private void drawText(Canvas canvas, PieInfoWrapper wrapper) {
        if (wrapper == null) return;

        //根据touch扩大量修正指示线和描述文字的位置
        float fixPos = (wrapper.equals(mTouchHelper.floatingWrapper) ? getFixTextPos(wrapper) : 0) + (wrapper.equals(mTouchHelper.lastFloatWrapper) ? getFixTextPos(wrapper) : 0);

        final float pointMargins = fixPos
                + pieRadius
                + mConfig.getGuideLineMarginStart()
                + (mConfig.isStrokeMode() ? mConfig.getStrokeWidth() / 2 : 0);
        float cx = (float) (pointMargins * Math.cos(Math.toRadians(wrapper.getMiddleAngle())));
        float cy = (float) (pointMargins * Math.sin(Math.toRadians(wrapper.getMiddleAngle())));

        //略

    }
```

求出了文字指引线的起始点，我们就清楚该文字属于哪个象限了，简单的判断x,y即可

```java
    private LineDirection calculateLineGravity(float startX, float startY) {
        if (startX > 0) {
            //在右边
            return startY > 0 ? LineDirection.BOTTOM_RIGHT : LineDirection.TOP_RIGHT;
        } else if (startX < 0) {
            //在左边
            return startY > 0 ? LineDirection.BOTTOM_LEFT : LineDirection.TOP_LEFT;
        } else if (startY == 0) {
            //刚好中间
            return startX > 0 ? LineDirection.CENTER_RIGHT : LineDirection.CENTER_LEFT;
        }
        return LineDirection.TOP_RIGHT;
    }
```

最后关于文字引导线的绘制，我们可以简单的PathMeasure使Path动态生长。

### 总结

对于甜甜圈，这个项目的主要难点在本篇已经说出，这个工程说复杂其实也不是很复杂，但说简单我个人认为也不能说分分钟完事，其实还是有挺多细节需要琢磨的。

诚然，这个库还是很有进步空间，比如issue里面所说的图例以及数据过多时的文字重叠等等（说起来，数据过多本就不适合饼图啊。。。。）。

但只要我还收到issue，我就不会放弃更新，一直迭代下去~

更多的话就不多说了，欢迎大家阅览项目：https://github.com/razerdp/AnimatedPieView

>同时也顺便推荐我的另一个项目：BasePopup：https://github.com/razerdp/BasePopup

欢迎交流哈-V-
