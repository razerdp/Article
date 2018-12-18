### 亲，还在为PopupWindow烦恼吗

***

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

BasePopup之所以 写成一个抽象类，除了更大程度的开放给开发者，更多的是让开发者更好地把功能内聚到PopupWindow中。

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

相比于封装相比您更关心其他的实现。

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

由于`dispatchKeyEvent`我们无法通过设置事件监听去拦截，而PopupWindow的`DecorView`又无法获取，看起来事件的分发进入了一个死胡同，然而通过细读源码，我们找到了一个突破口：WindowManager。

#### proxy WindowManager



