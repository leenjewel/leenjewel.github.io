---
layout: post
title: "纯 Java 代码实现 Android UI"
date: 2016-05-26 14:24:16 +0800
comments: true
categories: Android
---

这是一篇很初级也很简单的教程。

##为什么要用纯 Java 代码来实现 Android UI 界面

众所周知在 Android 开发时应用的 UI 界面一般是通过 XML 文件构建的。目前主流的 Android Studio 和 ~~Eclipse~~ 都可以通过鼠标拖拽控件的方式很高效的来搭建 UI 界面。那么为什么还要使用纯 Java 代码的方式来实现 UI 界面呢？

其实还是有一些特殊的场合需要使用这种纯 Java 代码的方式来实现 UI 界面的。例如 SDK 的开发。SDK 一般都是交付给第三方来使用的，要求接入流程尽可能简单，工作量尽可能少，最好直接一个 jar 包丢给对方，像这种情况纯 Java 代码来实现 UI 界面的方式就显得尤为重要了。

废话就到这，下面通过我工作中遇到的一个例子来展示一下如何用纯 Java 代码来实现 Android 的 UI 界面。

##Android UI 界面需求

<center>
<img width="320" height="640" src="http://7rfk33.com1.z0.glb.clouddn.com/tgsdk_sampleapp_screenshot.png" alt="Android UI 界面需求" />
</center>

如上图所示，我们需要使用纯 Java 代码来实现这样一个 UI 界面，它的要求是：

- 背景黑色半透明

- 居中一个占整个屏幕 76% 的 ImageView 用于显示图片

- 居中的 ImageView 要有一个白色的带圆角的边框

- 居中的 ImageView 右上角还要有一个圆形的关闭按钮

- 整个界面要求只能使用纯 Java 代码实现

- 除了 ImageView 中显示的图片以外，不能使用其他图片素材

我们一步一步按照要求用纯 Java 代码来实现这个 UI 界面。

##dp 与 px 的转换

至于 `dip`、`dp`、`px`、`sp`这些概念就不在这里介绍了，大家自行搜索。这里要注意的是，在使用 XML 做 UI 界面布局时一般会使用 `dp` 做单位，但是在纯 Java 实现中方法参数都使用 `px` 做单位，所以这里就牵扯到 `dp` 和 `px` 之间的转换。直接放出代码：

```java
    private int dp2px(float dp) {
        final float scale = getResources().getDisplayMetrics().density;  
        return (int) (dp * scale + 0.5f);
	}
```

##整体布局

这里就不具体介绍 Android 布局的相关知识了，直接说怎么做。这里用了线性布局 `LinearLayout`，具体布局方式是

- 先部署一个根布局 `LinearLayout`

- 在根布局中从上到下嵌套三个**横向** `LinearLayout`，其中头部和底部的 `layout_weight=12` 中间的 `layout_weight=76`

- 在上一步嵌套的中间的 `layout_weight=76` 的布局中从左到右再嵌套三个**纵向** `LinearLayout`，其中最左和最右的 `layout_weight=12` 中间的 `layout_weight=76`

这样我们就得到一块儿占屏幕 76% 居中放置的区域。对应的我们先展示出布局的 XML 形式：

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- 根布局 -->
<LinearLayout 
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/rootLayout"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:layout_weight="1">
    
    <!-- 占 12% 高度的顶部 -->
    <LinearLayout
    	android:id="@+id/topLayout"
    	android:orientation="horizontal"
    	android:layout_width="match_parent"
    	android:layout_height="0"
    	android:layout_weight="12">
    </LinearLayout>
    
    <!-- 占 76% 高度的中间 -->
    <LinearLayout
    	android:id="@+id/middleLayout"
    	android:orientation="horizontal"
    	android:layout_width="match_parent"
    	android:layout_height="0"
    	android:layout_weight="76">
    	
    	<!-- 占 12% 宽度的左边 -->
    	<LinearLayout
    		android:id="@+id/leftLayout"
    		android:orientation="vertical"
    		android:layout_width="0"
    		android:layout_height="match_parent"
    		android:layout_weight="12">
    	</LinearLayout>
    	
    	<!-- 占 76% 宽度的中间 -->
    	<LinearLayout
    		android:id="@+id/contentLayout"
    		android:orientation="vertical"
    		android:layout_width="0"
    		android:layout_height="match_parent"
    		android:layout_weight="76">
    	</LinearLayout>
    	
    	<!-- 占 12% 宽度的右边 -->
    	<LinearLayout
    		android:id="@+id/rightLayout"
    		android:orientation="vertical"
    		android:layout_width="0"
    		android:layout_height="match_parent"
    		android:layout_weight="12">
    	</LinearLayout>
    	
    </LinearLayout>
    
    <!-- 占 12% 高度的底部 -->
    <LinearLayout
    	android:id="@+id/bottomLayout"
    	android:orientation="horizontal"
    	android:layout_width="match_parent"
    	android:layout_height="0"
    	android:layout_weight="12">
    </LinearLayout>

</LinearLayout>
```

再用图片来示意一下：

<center>
<img width="320" height="640" src="http://7rfk33.com1.z0.glb.clouddn.com/tgsdk_sampleapp_screenshot_layout.png" alt="整体布局示意图" />
</center>

然后我们按照这个整体布局将它翻译成纯 Java 代码的样式

```java
        /* 根布局 */
        /* this 为当前的 Activity 实例 */
        LinearLayout rootLayout = new LinearLayout(this);
        LayoutParams rootLayoutParams = new LayoutParams(
            LayoutParams.MATCH_PARENT, 
            LayoutParams.MATCH_PARENT
        );
        rootLayout.setOrientation(LinearLayout.VERTICAL);
        rootLayout.setLayoutParams(rootLayoutParams);
        
        /* 占 12% 高度的顶部 */
        LinearLayout topLayout = new LinearLayout(this);
        topLayout.setOrientation(LinearLayout.HORIZONTAL);
        LinearLayout.LayoutParams vMarginLayoutParams = new LinearLayout.LayoutParams(
            LayoutParams.MATCH_PARENT, 
            0, 
            12.0f
        );
        topLayout.setLayoutParams(vMarginLayoutParams);
        rootLayout.addView(topLayout);
        
        /* 占 76% 高度的中间 */
        LinearLayout middleLayout = new LinearLayout(this);
        middleLayout.setOrientation(LinearLayout.HORIZONTAL);
        middleLayout.setLayoutParams(new LinearLayout.LayoutParams(
            LayoutParams.MATCH_PARENT, 
            0, 
            76.0f
        ));
        rootLayout.addView(middleLayout);
        
        /* 占 12% 宽度的左边 */
        LinearLayout leftLayout = new LinearLayout(this);
        leftLayout.setOrientation(LinearLayout.VERTICAL);
        LinearLayout.LayoutParams hMarginLayoutParams = new LinearLayout.LayoutParams(
            0, 
            LayoutParams.MATCH_PARENT, 
            12.0f);
        leftLayout.setLayoutParams(hMarginLayoutParams);
        middleLayout.addView(leftLayout);
        
        /* 占 76% 宽度的中间 */
        LinearLayout contentLayout = new LinearLayout(this);
        contentLayout.setOrientation(LinearLayout.VERTICAL);
        contentLayout.setLayoutParams(new LinearLayout.LayoutParams(
            0, 
            LayoutParams.MATCH_PARENT, 
            76.0f
        ));
        middleLayout.addView(contentLayout);
        
        /* 占 12% 宽度的右边 */
        LinearLayout rightLayout = new LinearLayout(this);
        rightLayout.setOrientation(LinearLayout.VERTICAL);
        rightLayout.setLayoutParams(hMarginLayoutParams);
        middleLayout.addView(rightLayout);
        
        /* 占 12% 高度的底部 */
        LinearLayout buttomLayout = new LinearLayout(this);
        buttomLayout.setOrientation(LinearLayout.HORIZONTAL);
        buttomLayout.setLayoutParams(vMarginLayoutParams);
        rootLayout.addView(buttomLayout);
        
        this.addContentView(rootLayout, rootLayoutParams);
```

这里确实没有什么难点可讲，自己参照 XML 布局文件和对应翻译出来的 Java 代码感受一下就能明白了。

## 关闭按钮

关闭按钮的难点是首先它是圆形按钮，不是一般的方形，其次上面还要画一个叉子，而这一切不允许用图片素材只能用纯代码来实现。我们还是先用 XML 来实现一遍然后再对应翻译成 Java 代码再实现一遍。

先看看这个关闭按钮的 XML 布局文件。大体思路是先创建一个圆形蓝色的背景形状，然后在按钮中应用这个形状来做背景即可。

背景形状的 XML 文件要放在 `res/drawable/` 目录下面，我们给起个名字 `res/drawable/close_button_background.xml` 内容如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_pressed="false">
        <shape android:shape="oval">
            <solid android:color="#1B81C9"/>
        </shape>
    </item>
    <item android:state_pressed="true">
        <shape android:shape="oval">
            <solid android:color="#1B81C9"/>
        </shape>
    </item>
</selector>
```

然后在 `res/layout/` 目录下的布局文件中定义关闭按钮，并使用上述的背景形状。

```xml
    <Button
        android:id="@+id/closeButton"
        android:layout_width="24dp"
        android:layout_height="24dp"
        android:background="@drawable/close_button_background"
        android:text="X"
        android:textColor="#ffffff" 
    />

```

上面的按钮布局代码只是一部分，具体关闭按钮的位置摆放方法我们放在后面来讲。用 XML 实现关闭按钮时并没有很好的处理上面白色的叉子的画法，当时只是“简单粗暴”的使用了按钮文字来解决，用了一个白色的大写 X 来作为关闭按钮上的叉子图形。不过接下来我们看到使用纯 Java 代码来实现时这里得到了完美的解决。

```java
        /* 关闭按钮 */
        /* this 是 Activity 实例 */
        int closeButtonSizePX = dp2px(24);
        
        Button closeButton = new Button(this);
        RelativeLayout.LayoutParams closeButtonLayoutParams = new RelativeLayout.LayoutParams(
            closeButtonSizePX, 
            closeButtonSizePX
        );
        closeButtonLayoutParams.addRule(RelativeLayout.ALIGN_PARENT_TOP);
        closeButtonLayoutParams.addRule(RelativeLayout.ALIGN_PARENT_RIGHT);
        closeButton.setLayoutParams(closeButtonLayoutParams);
        /* 按钮背景 */
        TGCPADCloseButtonBackground closeButtonBackground = new TGCPADCloseButtonBackground(this, closeButton);
        /* 圆形 */
        closeButtonBackground.setShape(GradientDrawable.OVAL);
        closeButtonBackground.setColor(Color.parseColor("#1B81C9"));
        closeButtonBackground.setStroke(12, Color.parseColor("#1B81C9"));
        closeButton.setBackground(closeButtonBackground);
```

上面代码中关于关闭按钮位置布局的部分可以先忽略，我们接下来会详细讲。这里我们把关注点放在按钮背景的实现上。这里关闭按钮的背景我们用了一个自定义的类，我们先看看这个自定义类的源码：

```java
class TGCPADCloseButtonBackground extends GradientDrawable {

	private Context _context = null;
	private View _view = null;
	
	public TGCPADCloseButtonBackground(Context context, View view) {
		_context = (context);
		_view = view;
	}
	
	private int dp2px(float dp) {
		final float density = _context.getResources().getDisplayMetrics().density;
		return (int) (dp * density + 0.5f);
	}
	
	@Override
	public void draw(Canvas canvas) {
		super.draw(canvas);
		Paint paint = new Paint();
		int width = _view.getLayoutParams().width;
		int height = _view.getLayoutParams().height;
		int margin = dp2px(8);
		paint.setColor(Color.WHITE);
		paint.setStyle(Style.STROKE);
		paint.setStrokeWidth(dp2px(2));
		canvas.drawLine(margin, margin, width-margin, height-margin, paint);
		canvas.drawLine(margin, height-margin, width-margin, margin, paint);
	}
}
```

这个自定义类的代码并不多，它继承自 [`GradientDrawable`](https://developer.android.com/reference/android/graphics/drawable/GradientDrawable.html) 这个类，按照官方文档上的说明这个类是用来绘制按钮和背景的

> A Drawable with a color gradient for buttons, backgrounds, etc.

所以这里的思路是让 `TGCPADCloseButtonBackground` 通过继承 `GradientDrawable` 来绘制一个蓝色的圆形按钮，然后再通过实现 `draw` 方法在已经绘制好的蓝色圆形按钮上面绘制一个白色的叉子。

##图片视图 ImageView

这里的难点在于绘制一个带圆角的外边框。不过有了上面绘制关闭按钮的思路，这里的实现方式也大同小异了。实现思路就是设置 `ImageView` 的内边距，然后给 `ImageView` 一个白色带圆角形状的背景，这样看上去就有了一个带圆角的边框。

这里就直接上 Java 代码了。

```java
        /* ImageView */
        /* this 是 Activity 实例 */
        ImageView adImageView = new ImageView(this);
        
        RelativeLayout.LayoutParams adImageLayoutParams = new RelativeLayout.LayoutParams(
            LayoutParams.MATCH_PARENT, 
            LayoutParams.MATCH_PARENT);
            
        /* 这里设置外边距为了和关闭按钮形成位置对齐 */
        int adImageMargin = dp2px(CLOSE_BUTTON_SIZE / 2);
        adImageLayoutParams.setMargins(
            adImageMargin, 
            adImageMargin, 
            adImageMargin, 
            adImageMargin);
        
        /* 这里设置内边距来实现白色边框 */
        int adImagePadding = dp2px(4);
        adImageView.setPadding(
            adImagePadding, 
            adImagePadding, 
            adImagePadding, 
            adImagePadding);
        adImageView.setLayoutParams(adImageLayoutParams);
        
        /* 白色带圆角形状的背景 */
        GradientDrawable adImageBackground = new GradientDrawable();
        adImageBackground.setShape(GradientDrawable.RECTANGLE);
        adImageBackground.setColor(Color.WHITE);
        adImageBackground.setCornerRadius(dp2px(6));
        adImageView.setBackground(adImageBackground);
```

##关闭按钮和图片视图的位置布局

通过整体布局我们已经得到了一块儿居中并占屏幕 76% 大小的区域。这里我们要做的是把图片视图和关闭按钮放置到这个区域里并保证关闭按钮的位置始终在图片视图的右上角。

这里我们的思路是在这块区域放置一个相对布局 `RelativeLayout`，然后把关闭按钮放进这个相对布局中并让关闭按钮的位置保持在布局的右上角固定不动。图片视图要有一个外边距，外边距的宽度正好是关闭按钮大小的一半，这样就使得关闭按钮的中心点正好和图片视图的右上角点重合。

老规矩先放出 XML 布局代码

```xml
    	<!-- 占 76% 宽度的中间 -->
    	<LinearLayout
    		android:id="@+id/contentLayout"
    		android:orientation="vertical"
    		android:layout_width="0"
    		android:layout_height="match_parent"
    		android:layout_weight="76">
    		
    		<RelativeLayout
    		    android:id="@+id/imageLayout"
    		    android:layout_width="match_parent"
    		    android:layout_height="match_parent">
    		    
    		    <ImageView
    		        android:id="@+id/adImage"
    		        android:layout_width="match_parent"
    		        android:layout_height="match_parent"
    		        android:layout_marginTop="12dp"
    		        android:layout_marginLeft="12dp"
    		        android:layout_marginRight="12dp"
    		        android:layout_marginBottom="12dp"
    		        android:paddingTop="4dp"
    		        android:paddingLeft="4dp"
    		        android:paddingRight="4dp"
    		        android:paddingBottom="4dp"
    		        android:background="@drawable/image_view_background"
    		    />
    		    
    		    <Button
    		        android:id="@+id/closeButton"
    		        android:layout_width="24dp"
    		        android:layout_heigth="24dp"
    		        android:layout_alignParentTop="true"
    		        android:layout_alignParentRight="true"
    		    />
    		    
    		</RelativeLayout>
    	</LinearLayout>
```

至于 Java 代码上面的章节已经都贴过了，这里就不再重复粘贴了，大家翻看上面的代码来感受一下吧。