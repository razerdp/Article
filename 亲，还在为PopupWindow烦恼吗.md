### 亲，还在为PopupWindow烦恼吗

***

>ps：预览图放到了文章最后

这篇文章其实想写很久了，然而一直以来总觉得BasePopup达不到自己的期望，所以也没有怎么去传播推荐，也因此一直都没有去写文章，直到最近狠下心重构2.0版本，并且完善了wiki等api文档后，才稍微满意了点，因此才开始着手写下这篇文章。

仓库地址：https://github.com/razerdp/BasePopup

相比于star，我更在乎您的issue~。

***

### 现状

跟不少产品经理、设计撕逼过的Android猿们应该都知道一件事：**没有什么交互，是不能用弹窗解决的，如果有，就弹多一个。**

诚然，如何榨干有限的屏幕空间同时又保持优雅的界面，是每个交互设计都要去思考的事情。

这时候，他们往往会选择一个神器：**弹窗**。

无论是从底部弹出还是从中间弹出，亦或是上往下弹右往左弹，甚至是弹出的时候带有动画，变暗，模糊等交互，在弹窗上的花样越来越多，而哭的，也往往是我们程序员。。。

在Android中，为了应付弹窗，我们可以选的东西其实挺多的：

 - Dialog
 - BottomSheetDialog
 - DialogFragment
 - PopupWindow
 - WindowManager直接怼入一个View
 - Dialog样式的Activity
 - 等等等等....
 
很多时候，我们都会选择Dialog而不选择PopupWindow，至于原因，很简单。。。PopupWindow好多坑！！！


### PopupWindow的优缺点

先说优点，相比于Dialog，PopupWindow的位置比较随意，可以在任意位置显示，而Dialog相对固定，其次就是背景变暗的效果，PopupWindow可以轻松的定制背景，无需复杂的黑科技。

而缺点，也有很多，这也是为什么大家更偏向于Dialog的原因，以下列举几条我认为最显著的缺点：
  - 创建复杂，与Dialog相比，每次都得写模板化的那几条初始化，很烦
  - 点击事件的蛋疼，要么无法响应backpress，要么点击外部不消失（**各个系统版本间的background问题**）
  - 系统版本的差异，每一次新系统的发布，都可以发现PopupWindow也悄悄的有所改动，而且更坑的是，往往在修复了旧的bug后，又引入了新的问题（**比如7.0高度match_parent时与以前显示不同的问题**）
  - PopupWindow内无法使用粘贴弹窗（**这个是固有问题，因为粘贴那个功能弹窗也是PopupWindow，而PopupWindow内的View是无法拿到windowToken的**）
  - 位置定位繁琐

为此，BasePopup库就诞生了。

### BasePopup解决方案

从1.0发布到现在2.1.1（准备发布2.1.2），为了开发BasePopup，走过的坑和读过的PopupWindow源码可以说是非常多了，当然，到现在为止，都还有一些坑没填，但BasePopup已经可以适配大多数情况了。

虽然这篇文章主要是推荐BasePopup，但更多的，是为了跟大家分享一下我的解决Idea，一直以来都是我一个人维护这个库，也没有多少人跟我交流其中的实现要点，在这里借这篇文章分享，同时也希望能得到更多人的建议或批评。

#### 创建复杂

首先我们看看普通的PopupWindow写法：

```java
//ps，以下三句其实都可以合并成一句在构造方法里，然而为了防止内容过长，这里分开写
PopupWindow popupWindow = new PopupWindow(this);
popupWindow.setWidth(ViewGroup.LayoutParams.WRAP_CONTENT);
popupWindow.setHeight(ViewGroup.LayoutParams.WRAP_CONTENT);
popupWindow.setContentView(LayoutInflater.from(this).inflate(R.layout.layout_popupwindow, null));
popupWindow.setBackgroundDrawable(new ColorDrawable(0x00000000));
popupWindow.setOutsideTouchable(false);
popupWindow.setFocusable(true);
```

虽然上面打了个注释解释道上面有几行是可以合并到同一个构造方法里解决，但PopupWindow有着5个以上的构造方法，即便有着IDE的自动提示，相信面对一大堆的构造方法依然是很头疼吧。

在BasePopup里，我们只需要继承**BasePopupWindow**并覆写`onCreateContentView`方法返回您的contentView即可，对于外部来说，只需要写两行甚至是一行代码就完成了。

```java
new DemoPopup(getContext()).showPopupWindow();
```
也许你会说，这不更蛋疼了么，为了一个PopupWindow，我不得不写多一个类。

这个问题就如MVP一样，为了更好地结构而不得不创建多一些类。。。

BasePopup之所以 写成一个抽象类，除了更大程度的开放给开发者，更多的是让开发者更好地把功能内聚到PopupWindow中，而不是去解决PopupWindow的各种蛋疼的坑。

当然，为了满足一些简单的PopupWindow实现而不希望又新建一个类，我们也提供了懒懒的方法支持链式使用：

```java
QuickPopupBuilder.with(getContext())
                .contentView(R.layout.popup_normal)
                .config(new QuickPopupConfig()
                        .gravity(Gravity.RIGHT | Gravity.CENTER_VERTICAL)
                        .withClick(R.id.tx_1, new View.OnClickListener() {
                            @Override
                            public void onClick(View v) {
                                Toast.makeText(getContext(), "clicked", Toast.LENGTH_LONG).show();
                            }
                        }))
                .show();
```

BasePopup是一个抽象类，具体实现交由子类（也就是开发者完成），同时也提供拦截器供开发者干预内部逻辑，最大化的开放自定义权限。

也许有更好的方法或设计模式，比如适配器等，这里就不细说了。

相比于封装相信您更关心其他的实现。

***

#### 事件消费

PopupWindow的事件一直都是让人头疼的事情，在6.0之前如果不设置background，那么是无法响应外部点击事件，而在6.0之后又修复了这一问题。

导致这一事情发生的，其实是跟PopupWindow内部的实现机制有关。

当我们给PopupWindow设置一个contentView的时候，这一个contentView其实是被PopupWindow内部的DecorView包裹住，而事件的响应则是由这个DecorView来分发。

在6.0之前，`PopupWindow#preparePopup()`源码如下：
```java
    private void preparePopup(WindowManager.LayoutParams p) {
		//忽略部分代码

        if (mBackground != null) {
            //忽略部分代码，当background不为空，才把contentView包裹进来
            PopupViewContainer popupViewContainer = new PopupViewContainer(mContext);
            PopupViewContainer.LayoutParams listParams = new PopupViewContainer.LayoutParams(
                    ViewGroup.LayoutParams.MATCH_PARENT, height
            );
            popupViewContainer.setBackground(mBackground);
            popupViewContainer.addView(mContentView, listParams);

            mPopupView = popupViewContainer;
        } else {
            mPopupView = mContentView;
        }
			//忽略后面代码
    }
```

而从6.0开始，preparePopup源码如下：
```java
    private void preparePopup(WindowManager.LayoutParams p) {
		//忽略部分代码
        if (mBackground != null) {
            mBackgroundView = createBackgroundView(mContentView);
            mBackgroundView.setBackground(mBackground);
        } else {
            mBackgroundView = mContentView;
        }
		//把contentView包裹到DecorView
        mDecorView = createDecorView(mBackgroundView);
        mDecorView.setIsRootNamespace(true);

        //忽略后面代码
    }
```

对于PopupWindow的事件，是在内部DecorView的`dispatchKeyEvent`和`onTouchEvent`方法里处理的，这里就不贴源码了。

由于`dispatchKeyEvent`我们无法通过设置事件监听去拦截，而PopupWindow的`DecorView`又无法获取，看起来事件的分发进入了一个死胡同，然而通过细读源码，我们找到了一个突破口：**WindowManager**。

#### proxy WindowManager

PopupWindow没有创建一个新的Window，它通过WindowManager添加一个新的View，其Type为`TYPE_APPLICATION_PANEL`，因此PopupWindow需要windowToken来作为依附。

在PopupWindow中，我们的contentView被包裹进DecorView，而DecorView则是通过WindowManager添加到界面中。

由于事件分发是在DecorView中，且没有监听器去拦截，**因此我们需要把这个DecorView再包多一层我们自定义的控件，然后添加到Window中**，这样一来，DecorView就成了我们的子类，对于事件的分发（甚至是measure/layout），我们就有了绝对的控制权，BasePopup正是这样做的。

然而，以上的步骤有个前提，就是如何代理掉WindowManager。（相当于寻找hook点）

在PopupWindow中，我们通过读源码可以获知，PopupWindow中的WindowManager是在两个地方被初始化：

 - 构造方法里
 - `setContentView()`
 
因此，我们也从这两个地方入手，继承PopupWindow并覆写以上两个方法，在里面通过反射来获取WindowManager并把它包裹到我们的`WindowManagerProxy`里面，然后再把我们的WindowManagerProxy设置给PopupWindow，这样就成功的偷天换日（代理）。

```java
abstract class BasePopupWindowProxy extends PopupWindow {
    private static final String TAG = "BasePopupWindowProxy";

    private BasePopupHelper mHelper;
    private WindowManagerProxy mWindowManagerProxy;

    //构造方法皆有调用init()，此处忽略其他构造方法

    public BasePopupWindowProxy(View contentView, int width, int height, boolean focusable, BasePopupHelper helper) {
        super(contentView, width, height, focusable);
        this.mHelper = helper;
        init(contentView.getContext());
    }

    void bindPopupHelper(BasePopupHelper mHelper) {
        if (mWindowManagerProxy == null) {
            tryToProxyWindowManagerMethod(this);
        }
        mWindowManagerProxy.bindPopupHelper(mHelper);
    }

    private void init(Context context) {
        setFocusable(true);
        setOutsideTouchable(true);
        setBackgroundDrawable(new ColorDrawable());
        tryToProxyWindowManagerMethod(this);
    }

    @Override
    public void setContentView(View contentView) {
        super.setContentView(contentView);
        tryToProxyWindowManagerMethod(this);
    }



    /**
     * 尝试代理掉windowmanager
     *
     * @param popupWindow
     */
    private void tryToProxyWindowManagerMethod(PopupWindow popupWindow) {
        if (mHelper == null || mWindowManagerProxy != null) return;
        PopupLogUtil.trace("cur api >> " + Build.VERSION.SDK_INT);
        troToProxyWindowManagerMethodBeforeP(popupWindow);
    }

   // android p 之后的代理，需要使用黑科技
    private void troToProxyWindowManagerMethodOverP(PopupWindow popupWindow) {
        try {
            WindowManager windowManager = PopupReflectionHelper.getInstance().getPopupWindowManager(popupWindow);
            if (windowManager == null) return;
            mWindowManagerProxy = new WindowManagerProxy(windowManager);
            PopupReflectionHelper.getInstance().setPopupWindowManager(popupWindow, mWindowManagerProxy);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // android p 之前的代理，普通反射即可
    private void troToProxyWindowManagerMethodBeforeP(PopupWindow popupWindow) {
        try {
            Field fieldWindowManager = PopupWindow.class.getDeclaredField("mWindowManager");
            fieldWindowManager.setAccessible(true);
            final WindowManager windowManager = (WindowManager) fieldWindowManager.get(popupWindow);
            if (windowManager == null) return;
            mWindowManagerProxy = new WindowManagerProxy(windowManager);
            fieldWindowManager.set(popupWindow, mWindowManagerProxy);
            PopupLogUtil.trace(LogTag.i, TAG, "尝试代理WindowManager成功");
        } catch (NoSuchFieldException e) {
            if (Build.VERSION.SDK_INT >= 27) {
                troToProxyWindowManagerMethodOverP(popupWindow);
            } else {
                e.printStackTrace();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```

说到反射，想必这里就有人觉得会不会存在性能问题，说实话，我当初也有这个顾虑，但实际上，从ART以来，反射的性能影响其实已经降低了很多，同时我们这里并非频繁的反射，所以在这一点上我认为可以忽略。

>另外反射获取WindowManager在Android P或以上并非在白名单中，因此BasePopup在这里通过`UnSafe`来绕过Api调用的控制，该方法参考[**android_p_no_sdkapi_support**](https://github.com/Guolei1130/android_p_no_sdkapi_support)，文章里总结了几种方法，本库采取最后一种，具体的这里就不细说了。


#### 系统版本的差异及其他问题

##### 位置控制

系统版本导致的位置问题很是让人头疼，在之前我通过一个类来适配api24之前，api24，以及api24之后，后来发现越写越多，因此产生了一个大胆的想法：

**PopupWindow的位置，我们自己来决定**

由于上面的代理，我们对PopupWindow的DecorView有着绝对的控制，所以由于系统版本导致PopupWindow显示的问题也很好解决。

对于PopupWindow的位置，因为DecorView是我们的自定义控件的子控件，因此在BasePopup中采取的方式是完全重写`onLayout()`。

我们的自定义控件是铺满整个屏幕的，因此我们针对DecorView进行layout，在视觉上的效果就是这个PopupWindow显示在了指定的位置上（**背景透明，而contentView是用户指定的xml，一般有颜色**），但实际上PopupWindow是铺满整个屏幕的。

>**（当然，对于普通的使用，也就PopupWindow不铺满整个屏幕也有适配）**

以下是layout的部分代码：
```java
    private void layoutWithIntercept(int l, int t, int r, int b) {
        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            if (child.getVisibility() == GONE) continue;
            int width = child.getMeasuredWidth();
            int height = child.getMeasuredHeight();

            int gravity = mHelper.getPopupGravity();

            int childLeft = child.getLeft();
            int childTop = child.getTop();

            int offsetX = mHelper.getOffsetX();
            int offsetY = mHelper.getOffsetY();

            boolean delayLayoutMask = mHelper.isAlignBackground();

            boolean keepClipScreenTop = false;

            if (child == mMaskLayout) {
                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            } else {
                boolean isRelativeToAnchor = mHelper.isShowAsDropDown();
                int anchorCenterX = mHelper.getAnchorX() + (mHelper.getAnchorViewWidth() >> 1);
                int anchorCenterY = mHelper.getAnchorY() + (mHelper.getAnchorHeight() >> 1);
                //不跟anchorView联系的情况下，gravity意味着在整个view中的方位
                //如果跟anchorView联系，gravity意味着以anchorView为中心的方位
                switch (gravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.LEFT:
                    case Gravity.START:
                        if (isRelativeToAnchor) {
                            childLeft = mHelper.getAnchorX() - width + childLeftMargin;
                        } else {
                            childLeft += childLeftMargin;
                        }
                        break;
                    case Gravity.RIGHT:
                    case Gravity.END:
                        if (isRelativeToAnchor) {
                            childLeft = mHelper.getAnchorX() + mHelper.getAnchorViewWidth() + childLeftMargin;
                        } else {
                            childLeft = getMeasuredWidth() - width - childRightMargin;
                        }
                        break;
                    case Gravity.CENTER_HORIZONTAL:
                        if (isRelativeToAnchor) {
                            childLeft = mHelper.getAnchorX();
                            offsetX += anchorCenterX - (childLeft + (width >> 1));
                        } else {
                            childLeft = ((r - l - width) >> 1) + childLeftMargin - childRightMargin;
                        }
                        break;
                    default:
                        if (isRelativeToAnchor) {
                            childLeft = mHelper.getAnchorX() + childLeftMargin;
                        }
                        break;
                }

                switch (gravity & Gravity.VERTICAL_GRAVITY_MASK) {
                    case Gravity.TOP:
                        if (isRelativeToAnchor) {
                            childTop = mHelper.getAnchorY() - height + childTopMargin;
                        } else {
                            childTop += childTopMargin;
                        }
                        break;
                    case Gravity.BOTTOM:
                        if (isRelativeToAnchor) {
                            keepClipScreenTop = true;
                            childTop = mHelper.getAnchorY() + mHelper.getAnchorHeight() + childTopMargin;
                        } else {
                            childTop = b - t - height - childBottomMargin;
                        }
                        break;
                    case Gravity.CENTER_VERTICAL:
                        if (isRelativeToAnchor) {
                            childTop = mHelper.getAnchorY() + mHelper.getAnchorHeight();
                            offsetY += anchorCenterY - (childTop + (height >> 1));
                        } else {
                            childTop = ((b - t - height) >> 1) + childTopMargin - childBottomMargin;
                        }
                        break;
                    default:
                        if (isRelativeToAnchor) {
                            keepClipScreenTop = true;
                            childTop = mHelper.getAnchorY() + mHelper.getAnchorHeight() + childTopMargin;
                        } else {
                            childTop += childTopMargin;
                        }
                        break;
                }

                int left = childLeft + offsetX;
                int top = childTop + offsetY + (mHelper.isFullScreen() ? 0 : -getStatusBarHeight());
                int right = left + width;
                int bottom = top + height;

                //针对clipToScreen和autoLocated的情况，这里因篇幅限制忽略
                }
                child.layout(left, top, right, bottom);
                if (delayLayoutMask) {
                    mMaskLayout.handleAlignBackground(left, top, right, bottom);
                }
            }

        }
    }
```

对于layout，我们只需要区分PopupWindow是否跟anchorView关联，然后根据Gravity和Offset进行位置的计算。

这些操作对于经常自定义控件的同学来说简直就是拈手即来。

而对于平时的PopupWindow用法，即PopupWindow不铺满整个屏幕，在BasePopup中则是跟普通用法一样计算offset。

```java
    private void onCalculateOffsetAdjust(Point offset, boolean positionMode, boolean relativeToAnchor) {
        int leftMargin = 0;
        int topMargin = 0;
        int rightMargin = 0;
        int bottomMargin = 0;
        if (mHelper.getParaseFromXmlParams() != null) {
            leftMargin = mHelper.getParaseFromXmlParams().leftMargin;
            topMargin = mHelper.getParaseFromXmlParams().topMargin;
            rightMargin = mHelper.getParaseFromXmlParams().rightMargin;
            bottomMargin = mHelper.getParaseFromXmlParams().bottomMargin;
        }
        //由于showAsDropDown系统已经帮我们定位在view的下方，因此这里的offset我们仅需要做微量偏移
        switch (getPopupGravity() & Gravity.HORIZONTAL_GRAVITY_MASK) {
            case Gravity.LEFT:
            case Gravity.START:
                if (relativeToAnchor) {
                    offset.x += -getWidth() + leftMargin;
                } else {
                    offset.x += leftMargin;
                }
                break;
            case Gravity.RIGHT:
            case Gravity.END:
                if (relativeToAnchor) {
                    offset.x += mHelper.getAnchorViewWidth() + leftMargin;
                } else {
                    offset.x += getScreenWidth() - getWidth() - rightMargin;
                }
                break;
            case Gravity.CENTER_HORIZONTAL:
                if (relativeToAnchor) {
                    offset.x += (mHelper.getAnchorViewWidth() - getWidth()) >> 1;
                } else {
                    offset.x += ((getScreenWidth() - getWidth()) >> 1) + leftMargin - rightMargin;
                }
                break;
            default:
                if (!relativeToAnchor) {
                    offset.x += leftMargin;
                }
                break;
        }

        switch (getPopupGravity() & Gravity.VERTICAL_GRAVITY_MASK) {
            case Gravity.TOP:
                if (relativeToAnchor) {
                    offset.y += -(mHelper.getAnchorHeight() + getHeight()) + topMargin;
                } else {
                    offset.y += topMargin;
                }
                break;
            case Gravity.BOTTOM:
                //系统默认就在下面.
                if (!relativeToAnchor) {
                    offset.y += getScreenHeight() - getHeight() - bottomMargin;
                }
                break;
            case Gravity.CENTER_VERTICAL:
                if (relativeToAnchor) {
                    offset.y += -((getHeight() + mHelper.getAnchorHeight()) >> 1);
                } else {
                    offset.y += ((getScreenHeight() - getHeight()) >> 1) + topMargin - bottomMargin;
                }
                break;
            default:
                if (!relativeToAnchor) {
                    offset.y += topMargin;
                }
                break;
        }
```

正因为位置有我们来控制，所以不仅仅在所有版本中统一了位置的计算方式，而且更重要的是，PopupWindow的`Gravity`这一个属性被充分使用，再也不用去计算心塞的偏移量了。

举个例子，比如我们要显示在某个 view的右边，同时自己跟他垂直对齐。

在系统的PopupWindow中，你可能要这么写：

```java

//前面忽略创建方法
popup.showAsDropDown(v,v.getWidth(),-(v.getHeight()+popup.getHeight())>>1)
```

上面的代码还是比较简单的，popup默认显示在anchorView的下方，此处需要计算偏移量，使popup可以偏移到view的右方，但是有个值得关注的是popup在显示之前是获取不到正确的contentView的宽高的。


而在BasePopup中，你要写的，仅仅是这样：

```java

//前面忽略创建方法
popup.setPopupGravity(Gravity.RIGHT|Gravity.CENTER_VERTICAL);
popup.showPopupWindow(anchorView);
```

在BasePopup中，因为layout由我们接管，因此在onLayout中我们其实是知道contentView的宽高，因此根据上面的代码，我们直接通过Gravity来计算出Popup的正确位置即可。

**关于Gravity的Demo**：

<img src="https://github.com/razerdp/Pics/blob/master/BasePopup/demo_gravity.gif" height="480"/>



##### 背景模糊

同时我们可以针对这个自定义的ViewGroup默认添加背景，在BasePopup中，背景添加了一个ImageView和一个View，分别处理模糊和背景颜色。

其中背景的模糊采取的是RenderScript，对于不支持的情况则采取fastBlur，由于模糊基本上大同小异，在这里就不贴代码了。


#### 其他问题

到目前位置，BasePopup满足多数的PopupWindow使用，但仍然有不足，比如没有支持PopupWindow的update()方法，因为我们多数时候PopupWindow都是展示用，而且基本上都是展示一次后就消掉。

但不排除有PopupWindow跟随某个View而更新自己的位置这一需求，因此在接下来的维护里，这个问题将会纳入到之后的工作中。

最后感谢提issue的小伙伴们，你们的每一个issue我都认真的看且有空就去清掉。

最后的最后，希望本文能对看到这篇文章的你有些帮助~

thanks

仓库地址：https://github.com/razerdp/BasePopup

***

**18/12/19：candy版本更新到2.1.3-alpha，已经支持update~感谢支持**

预览图：

| **anchorView绑定** | **不同方向弹出** |
| - | - |
| <img src="https://github.com/razerdp/Pics/blob/master/BasePopup/wiki/linkto/linkto.gif" height="480"/> | <img src="https://github.com/razerdp/Pics/blob/master/BasePopup/demo_locatepopup.gif" height="480"/> |
| **任意位置显示**  | **参考anchorView更新** |
| <img src="https://github.com/razerdp/Pics/blob/master/BasePopup/wiki/anypos/anypos.gif" height="480"/> | <img src="https://github.com/razerdp/Pics/blob/master/BasePopup/wiki/update/update.gif" height="480"/> |
| **从下方弹出并模糊背景**  | **朋友圈评论弹窗** |
| <img src="https://github.com/razerdp/Pics/blob/master/BasePopup/demo_blur_from_bottom.gif" height="480"/> | <img src="https://github.com/razerdp/Pics/blob/master/BasePopup/demo_comment.gif" height="480"/> |





