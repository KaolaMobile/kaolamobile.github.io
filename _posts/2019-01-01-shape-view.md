---
layout:     post
title:      "ShapeView 实现原理"
subtitle:   ""
date:       2019-01-02
author:     "许方镇"
header-img: "img/bg14.jpg"
tags:
    Android View
---

[ShapeView](http://note.youdao.com/noteshare?id=eafd8f6ee47014fe7a9ececaa1d570d5&sub=E215AE0C23AD46ADA283A2C7CEC6BCD9)使用教程

### 简介
ShapeView本质是增加shape标签中对应的属性，使其能生成shape相关的drawable当背景<br>
其中包括有两个助手类，ShapeHelper 和 MaskHelper。这两个类分别实现了以下功能

* ShapeHelper 
    * 实现shape标签中shape="rectangle"的corners、solid、stroke、gradient功能
* MaskHelper
    * 绘制圆角蒙层、圆角镂空
    * 实现绘制分割线
    * 实现绘制阴影


### ShapeHelper

ShapeHelper中主要是生成一张KluiShapeDrawable背景图，通过setBackground对View进行设置。<br>
*由此可以看出shapeView在xml中设置的``` android:background ```将会被丢弃*

#### KluiShapeDrawable的生成过程
1. 首先需要知道，KluiShapeDrawable是继承自GradientDrawable，而GradientDrawable正是源码中shape标签生成的背景图<br><br>
源码如下：

```
tv.setBackground(etResources().getDrawable(R.drawable.corner_max_stroke_fff_1dp));
```

从上面的代码中可以看出，getDrawable是shape转GradientDrawable的一个入口,
查看getDrawable的源码，发现调用了getDrawableForDensity，该方法又调用了ResourcesImpl类中的loadDrawable
查看该方法下的关键代码

```java
if (cs != null) {
    if (TRACE_FOR_DETAILED_PRELOAD) {
        // Log only framework resources
        if (((id >>> 24) == 0x1) && (android.os.Process.myUid() != 0)) {
            final String name = getResourceName(id);
            if (name != null) {
                Log.d(TAG_PRELOAD, "Hit preloaded FW drawable #"
                        + Integer.toHexString(id) + " " + name);
            }
        }
    }
    dr = cs.newDrawable(wrapper);
} else if (isColorDrawable) {
    dr = new ColorDrawable(value.data);
} else {
    dr = loadDrawableForCookie(wrapper, value, id, density, null);
}

```
在没有cs(静态状态)、不是颜色图片的情况下，走进了loadDrawableForCookie，如果结尾是xml，则使用createFromXml

```java
if (file.endsWith(".xml")) {
    final XmlResourceParser rp = loadXmlResourceParser(
            file, id, value.assetCookie, "drawable");
    dr = Drawable.createFromXmlForDensity(wrapper, rp, density, theme);
    rp.close();
}

```
createFromXmlForDensity调用了createFromXmlInnerForDensity

```java
public static Drawable createFromXmlForDensity(@NonNull Resources r,
        @NonNull XmlPullParser parser, int density, @Nullable Theme theme)
        throws XmlPullParserException, IOException {
    AttributeSet attrs = Xml.asAttributeSet(parser);
    ……
    Drawable drawable = createFromXmlInnerForDensity(r, parser, attrs, density, theme);
    ……
    return drawable;
}

```
之后调用了DrawableInflater的inflateFromXmlForDensity方法

```java
static Drawable createFromXmlInnerForDensity(@NonNull Resources r,
            @NonNull XmlPullParser parser, @NonNull AttributeSet attrs, int density,
            @Nullable Theme theme) throws XmlPullParserException, IOException {
    return r.getDrawableInflater().inflateFromXmlForDensity(parser.getName(), parser, attrs,
            density, theme);
}

```
然后又调用了DrawableInflater的inflateFromTag

```java
Drawable inflateFromXmlForDensity(@NonNull String name, @NonNull XmlPullParser parser,
            @NonNull AttributeSet attrs, int density, @Nullable Theme theme)
            throws XmlPullParserException, IOException {
    ……
    Drawable drawable = inflateFromTag(name);
    ……
    drawable.inflate(mRes, parser, attrs, theme);
    ……
    return drawable;
}

```
inflateFromTag的方法明显看到返回了GradientDrawable

```java
@NonNull
@SuppressWarnings("deprecation")
private Drawable inflateFromTag(@NonNull String name) {
    switch (name) {
    ……
        case "color":
            return new ColorDrawable();
        case "shape":
            return new GradientDrawable();
        case "vector":
            return new VectorDrawable();
    ……
        default:
            return null;
    }
}
```

2. 调用fromAttributeSet依次设置corners、stroke、solid、gradient<br><br>
 
* corners<br>
优先级：如果设置了任意一个角度，则不使用全局角度。如果设置了全局角度，则不使用设置半圆的属性

```java
if (mTopLeftRadius > 0 || mTopRightRadius > 0 || mBottomLeftRadius > 0 || mBottomRightRadius > 0) {
            float[] radii = new float[] {
            mTopLeftRadius, mTopLeftRadius,
            mTopRightRadius, mTopRightRadius,
            mBottomRightRadius, mBottomRightRadius,
            mBottomLeftRadius, mBottomLeftRadius
    };
    shapeDrawable.setCornerRadii(radii);
} else {
    int mRadius = typedArray.getDimensionPixelSize(R.styleable.ShapeView_cornersRadius, 0);
    if (mRadius > 0) {
        shapeDrawable.setCornerRadius(mRadius);
    } else {
        shapeDrawable.setIsSemicircle(typedArray.getBoolean(R.styleable.ShapeView_isSemicircle, false));
    }
}

```
到view的size变化会触发onBoundsChange，在此处设置是否需要设置成半圆

```java
@Override
protected void onBoundsChange(Rect r) {
    super.onBoundsChange(r);
    if (mIsSemicircle) {
        setCornerRadius(Math.min(r.width(), r.height()) / 2);
    }
}

```

* stroke

```java
if (typedArray.hasValue(R.styleable.ShapeView_strokeColor)) {
    ColorStateList strokeColor = typedArray.getColorStateList(R.styleable.ShapeView_strokeColor);
    int strokeWidth = typedArray.getDimensionPixelSize(R.styleable.ShapeView_strokeWidth, 0);
    int strokeDashWidth = typedArray.getDimensionPixelSize(R.styleable.ShapeView_strokeDashWidth, 0);
    int strokeDashGap = typedArray.getDimensionPixelSize(R.styleable.ShapeView_strokeDashGap, 0);
    shapeDrawable.setStrokeData(strokeWidth, strokeColor, strokeDashWidth, strokeDashGap);
}

```

```java
public void setStrokeData(int width, @Nullable ColorStateList colors, int dashWidth, int dashGap) {
    //大于等于5.0，支持ColorStateList，直接通过GradientDrawable的setStroke设置
    if (hasNativeStateListAPI()) {
    ……
            setStroke(width, colors, dashWidth, dashGap);
    ……
    } else {
    ……
        //小于5.0，获取状态对应的color，通过GradientDrawable的setStroke设置
        final int currentColor;
    ……
            currentColor = colors.getColorForState(getState(), 0);
    ……
            setStroke(width, currentColor, dashWidth, dashGap);
    ……
    }
}

```

* solid

 ```java
if (typedArray.hasValue(R.styleable.ShapeView_solidColor)) {
    shapeDrawable.setSolidData(typedArray.getColorStateList(R.styleable.ShapeView_solidColor));
}

```
```java
public void setSolidData(@Nullable ColorStateList colors) {
    //大于等于5.0，支持ColorStateList，直接调用GradientDrawable的setColor
    if (hasNativeStateListAPI()) {
        super.setColor(colors);
    } else {
        //小于5.0，获取状态对应的color，通过调用GradientDrawable的setColor
            ……
        final int currentColor;
            ……
            currentColor = colors.getColorForState(getState(), 0);
            ……
        setColor(currentColor);
    }
}

```
* gradient

```java
int gradientStartColor = typedArray.getColor(R.styleable.ShapeView_gradientStartColor, 0);
int gradientEndColor = typedArray.getColor(R.styleable.ShapeView_gradientEndColor, 0);
if (gradientStartColor != 0 || gradientEndColor != 0) {
    int gradientType = typedArray.getInt(R.styleable.ShapeView_gradientType, 0);

    float gradientCenterX = getFloatOrFraction(typedArray, R.styleable.ShapeView_gradientCenterX, 0.5f);
    float gradientCenterY = getFloatOrFraction(typedArray, R.styleable.ShapeView_gradientCenterY, 0.5f);
    int gradientAngle = (int) typedArray.getFloat(R.styleable.ShapeView_gradientAngle, 0);

    boolean hasCenterColor = typedArray.hasValue(R.styleable.ShapeView_gradientCenterColor);
    int gradientCenterColor = typedArray.getColor(R.styleable.ShapeView_gradientCenterColor, 0);
    //设置渐变类型
    shapeDrawable.setGradientType(gradientType);
    shapeDrawable.setGradientCenter(gradientCenterX, gradientCenterY);
    //通过GradientDrawable的setColors设置渐变色
    if (hasCenterColor) {
        shapeDrawable.setColors(new int[] { gradientStartColor, gradientCenterColor, gradientEndColor });
    } else {
        shapeDrawable.setColors(new int[] { gradientStartColor, gradientEndColor });
    }
    //通过GradientDrawable的setOrientation设置渐变方向
    if (gradientType == LINEAR_GRADIENT) {
        shapeDrawable.setOrientation(getOrientation(gradientAngle));
    } else if (gradientType == RADIAL_GRADIENT) {
        shapeDrawable.setGradientRadius(
                typedArray.getDimensionPixelSize(R.styleable.ShapeView_gradientRadius, 0));
    }
    //如果为true，则可在LevelListDrawable中使用。这通常应为“false”，否则形状不会显示。
    shapeDrawable.setUseLevel(typedArray.getBoolean(R.styleable.ShapeView_gradientUseLevel, false));
}

```

### MaskHelper 
MaskHelper主要是通过draw方法绘制相应的形状盖在上层，其中用到的属性见[使用教程](http://note.youdao.com/noteshare?id=eafd8f6ee47014fe7a9ececaa1d570d5&sub=E215AE0C23AD46ADA283A2C7CEC6BCD9)


我们知道draw方法是一个view开始绘制的入口，
内部执行了以下的几步

```xml
/*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */
```

忽略第二和第五步、绘制过程如下

1. 绘制背景
2. 绘制内容
3. 绘制子view
4. 绘制装饰品（比如滚动条）

而复写了ShapeView中的draw如下

```
@Override
    public void draw(Canvas canvas) {
        if (mMaskHelper != null) {
            mMaskHelper.beforeDraw(canvas);
        }
        super.draw(canvas);
        if (mMaskHelper != null) {
            mMaskHelper.drawOver(canvas);
        }
    }
```

因此是在绘制开始前调用了beforeDraw，绘制结束后调用了drawOver<br>

#### drawOver
我们知道越晚绘制的东西越在上层，因此drawOver里绘制的东西是可以盖住之前绘制的内容。接下来看如何绘制圆角。<br>
首先在构造函数中调用init，获取各个属性，初始化画笔等<br>

1.在drawOver中首先绘制的就是圆角，通过canvas.drawPath绘制圆角的生成的路径mMaskPathList

```java
public void drawOver(Canvas canvas) {
        ……
        //绘制圆角
        if (mMaskColor != 0) {
            mPaint.setColor(mMaskColor);
            mPaint.setStyle(Paint.Style.FILL);
            if (!KluiUtils.isCollectionEmpty(mMaskPathList)) {
                ……
                for (Path path : mMaskPathList) {
                    canvas.drawPath(path, mPaint);
                }
                ……
            }
        }
            ……
    }
```

在onSizeChanged生成圆角路径

```
public void onSizeChanged(int w, int h, int oldw, int oldh) {
        mWidth = w;
        mHeight = h;
        mMaskPathList.clear();
        if (mMaskTopLeftRadius > 0) {
            mMaskTopLeftPath.rewind();
            mMaskTopLeftPath.moveTo(mMaskPaddingLeft - 1, mMaskPaddingTop - 1);
            mMaskArcRectF.set(mMaskPaddingLeft, mMaskPaddingTop,
                    (mMaskTopLeftRadius + mInsideStrokeWidth) * 2 + mMaskPaddingLeft,
                    (mMaskTopLeftRadius + mInsideStrokeWidth) * 2 + mMaskPaddingTop);
            mMaskTopLeftPath.arcTo(mMaskArcRectF, 270, -90);
            mMaskPathList.add(mMaskTopLeftPath);
        }
            ……
    }
```
首先通过rewind清除路径上的数据
>reset清除path上的内容，重置path到 path = new Path()的初始状态。<br>
>rewind清除path上的内容，但会保留path上相关的数据结构，以高效的复用。

之后通过moveTo绘制圆角蒙层的顶点，通过arcTo绘制圆角蒙层的弧线，即完成了一个圆角蒙层的路径，左上、右上、左下、右下分别添加进入mMaskPathList即可。

这里推荐一篇[path](https://blog.csdn.net/jsagacity/article/details/81741175)相关文章


2.绘制完圆角，绘制轮廓，如果轮廓是圆角矩形，则绘制drawRoundRect，否则drawPath
Path的生成是通过矩形+圆角的方式addRoundRect

```java
//绘制内部轮廓
if (mInsideStrokeColor != 0 && mInsideStrokeWidth > 0) {
    ……
    if (mMaskRadius != 0) {
        canvas.drawRoundRect(getInsideRect((mInsideStrokeWidth + 1) / 2), mMaskRadius, mMaskRadius, mPaint);
    } else {
        mInsidePath.rewind();
        mInsidePath.addRoundRect(getInsideRect((mInsideStrokeWidth + 1) / 2), mShadowRadii, Path.Direction.CCW);
        canvas.drawPath(mInsidePath, mPaint);
    }
}
```

3.最后绘制阴影
不等不说，绘制阴影的方法很原始，但是确实有用，就是一圈一圈画出矩形，每圈矩形颜色alpha值渐变

```java
private void drawShadow(Canvas canvas) {
    if (mShadowColor == 0) {
        return;
    }
    canvas.clipPath(getShadowWrapPath());
    mPaint.setStrokeWidth(HEIGHT_ALPHA_SHADOW_WIDTH);
    mPaint.setStyle(Paint.Style.STROKE);
    for (int i = mMaxMaskPadding; i > 0; i -= HEIGHT_ALPHA_SHADOW_WIDTH * 0.5f) {
        mPaint.setColor(
                Color.argb((mShadowAlphaToHex + mShadowAlphaOffset * i / mMaxMaskPadding), mColorR, mColorG,
                        mColorB));
        mShadowRect.set(i, i, mWidth - i, mHeight - i);
        canvas.drawRoundRect(mShadowRect, mMaskRadius, mMaskRadius, mPaint);
    }
}
```

clipPath：截取绘制区域

getShadowWrapPath() 获取阴影绘制区域。通过 顺时针CW 和 逆时针CCW 的矩形围成一个封闭的矩形环

```java
private Path getShadowWrapPath() {
    mShadowWrapPath.rewind();
    mShadowWrapPath.addRect(0, 0, mWidth, mHeight, Path.Direction.CW);
    mShadowRect.set(mMaskPaddingLeft, mMaskPaddingTop, mWidth - mMaskPaddingRight, mHeight - mMaskPaddingBottom);
    mShadowWrapPath.addRoundRect(mShadowRect, mShadowRadii, Path.Direction.CCW);
    return mShadowWrapPath;
}
```
#### beforeDraw

beforeDraw主要是为了实现镂空效果 mIsHollow 和 绘制最底层的背景色mInsideSolidColor

```java
public void beforeDraw(Canvas canvas) {
    if (mIsHollow) {
        canvas.saveLayer(0, 0, mWidth, mHeight, null, Canvas.ALL_SAVE_FLAG);
        mMaskColor = Color.WHITE;
    }
    if (mInsideSolidColor != 0) {
        mPaint.setColor(mInsideSolidColor);
        mPaint.setStyle(Paint.Style.FILL);
        mInsidePath.rewind();
        mInsidePath.addRoundRect(getInsideRect(mInsideStrokeWidth), mShadowRadii, Path.Direction.CCW);
        canvas.drawPath(mInsidePath, mPaint);
    }
}
```

绘制最底层的背景色mInsideSolidColor比较简单，通过drawPath即可

镂空

主要通过saveLayer和[setXfermode](https://developer.android.com/reference/android/graphics/PorterDuff.Mode)实现

1. 通过saveLayer生成一张离屏的bitmap，因此比较耗内存
>This behaves the same as save(), but in addition it allocates 
andredirects drawing to an offscreen bitmap。 this method is very 
expensive, incurring more than double rendering cost for contained 
content. Avoid using this method, especially if the bounds provided 
are large

2. 绘制背景\ 绘制内容\绘制子view\ 绘制装饰品
3. 设置setXfermode，是其与离屏的bitma进行合成(抠掉)

```java
public void drawOver(Canvas canvas) {
    ……
    //绘制圆角
    if (mMaskColor != 0) {
           ……
        if (!KluiUtils.isCollectionEmpty(mMaskPathList)) {
            if (mIsHollow) {
                mPaint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_OUT));
            }
            for (Path path : mMaskPathList) {
                canvas.drawPath(path, mPaint);
            }
            if (mIsHollow) {
                canvas.restore();
                mPaint.setXfermode(null);
            }
        }
    }
    ……
}
```

### 问题
为什么不用clipPath设置一个区域进行不支持圆角<br>
答：对于圆角部分，会有明显锯齿；对于部分圆角的情况，只有高版本支持

为什么需要先saveLayer，不保存一个图层会怎样<br>
答：会把该view的父view也抠掉

### 扩展
如何对普通的TextView设置shape的属性后，让其支持各种背景<br>
下文给出答案<br>
[Android技能树 — LayoutInflater Factory小结](https://www.jianshu.com/p/8d8ada21ab82)