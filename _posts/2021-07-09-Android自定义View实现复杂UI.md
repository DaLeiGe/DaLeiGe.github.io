---
layout: post
title: "Android自定义View实现复杂UI"
date: 2021-07-01 13:00:20.000000000 +09:00
categories: [Android, UI]
tags: [Android]
typora-root-url: ..
---
<meta name="referrer" content="no-referrer"/>

## 一、背景
#### 1.1、控件效果
要实现的自定义控件效果大致如下，实现过程中用到了比较多的自定义View的API，觉得比较有代表性，就分享出来也当做学习总结
项目代码已上传github
[https://github.com/DaLeiGe/AndroidSamples/tree/master/ProgressView](https://github.com/DaLeiGe/AndroidSamples/tree/master/ProgressView)

![升高温度](https://upload-images.jianshu.io/upload_images/7312294-fb47c26560a3bcf2.gif?)

![绿色环倒计时](https://upload-images.jianshu.io/upload_images/7312294-ae2b02797d833ace.gif?imageMogr2/auto-orient/strip)

![1595899207(1).jpg](https://upload-images.jianshu.io/upload_images/7312294-3bae20a83533965b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 1.2、从功能上分析一下这个控件，大致有以下特点
- 随机运动粒子从圆周向圆心运动，并与切线方向有正负30°的角度差，粒子透明度、半径、运动速度随机，运动超过一定距离或者时间消失
- 背景圆有一个从内到外的渐变色
- 计时模式下圆环有一个颜色渐变的顺时针rotate动画
- 整个背景圆颜色随着扇形角度变化而变化
- 指针颜色变化
- 数字变化是上下切换动画

#### 1.3、从结构上分析
这个控件可以拆分为两个部分，由背景圆+数字控件两个部分构成的组合控件，之所以把数字控件单独拆分出来，也是为了方便做数字上下跳动的动画，毕竟通过控制drawText的位置实现动画感觉不方便，直接通过View的属性动画更好实现

![未命名表单.jpg](https://upload-images.jianshu.io/upload_images/7312294-3c8429b3b210b8d7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 二、 背景圆实现
#### 2.1、实现粒子运动
使用AnimPoint.java表示运动粒子，它具有x,y坐标，半径，角度，运动速度，透明度等属性，通过这些属性就可以画出一个静态的粒子
```java
public class AnimPoint implements Cloneable {
    /**
     * 粒子原点x坐标
     */
    private float mX;
    /**
     * 粒子原点y坐标
     */
    private float mY;
    /**
     * 粒子半径
     */
    private float radius;
    /**
     * 粒子初始位置的角度
     */
    private double anger;
    /**
     * 一帧移动的速度
     */
    private float velocity;
    /**
     * 总共移动的帧数
     */
    private int num = 0;

    /**
     * 透明度 0~255
     */
    private int alpha = 153;

    /**
     * 随机偏移角度
     */
    private double randomAnger = 0;
}
```
粒子的初始位置位于随机角度的圆周，且一个粒子具有随机的半径，透明度，速度等，通过init()方法，实现初始化粒子如下
```java
public void init(Random random, float viewRadius) {
        anger = Math.toRadians(random.nextInt(360));
        velocity = random.nextFloat() * 2F;
        radius = random.nextInt(6) + 5;
        mX = (float) (viewRadius * Math.cos(anger));
        mY = (float) (viewRadius * Math.sin(anger));
        //随机偏移角度-30°~30°
        randomAnger = Math.toRadians(30 - random.nextInt(60));
        alpha = 153 + random.nextInt(102);
    }
```
想让粒子运动起来，使用update更新粒子的这些坐标属性就能实现，比如粒子现在坐标在(5,5)，通过update()改变粒子的坐标到(6,6),结合属性动画不停地调用update()则就能不停的改变x,y的坐标，实现粒子运动，然后当粒子移动超过一定距离，或者调用update超过一定次数，再重新调用init()让粒子重新从圆周上开始下一个生命周期运动
```java
public void updatePoint(Random random, float viewRadius) {
        //每一帧偏移的像素大小
        float distance = 1F;
        double moveAnger = anger + randomAnger;
        mX = (float) (mX - distance * Math.cos(moveAnger) * velocity);
        mY = (float) (mY - distance * Math.sin(moveAnger) * velocity);
        //模拟半径逐渐变小
        radius = radius - 0.02F * velocity;
        num++;
        //如果到了最大值 则重新给运动粒子一个轨迹属性
        int maxDistance = 180;
        int maxNum = 400;
        if (velocity * num > maxDistance || num > maxNum) {
            num = 0;
            init(random, viewRadius);
        }
    }
```
在View中大致实现如下
```java
/**
     * 初始化动画
     */
    private void initAnim() {
        //绘制运动的粒子
        AnimPoint mAnimPoint = new AnimPoint();
        for (int i = 0; i < pointCount; i++) {
            //通过clone创建对象，避免重复创建
            AnimPoint cloneAnimPoint = mAnimPoint.clone();
            //先给每个粒子初始化各类属性
            cloneAnimPoint.init(mRandom, mRadius - mOutCircleStrokeWidth / 2F);
            mPointList.add(cloneAnimPoint);
        }
        //画运动粒子
        mPointsAnimator = ValueAnimator.ofFloat(0F, 1F);
        mPointsAnimator.setDuration(Integer.MAX_VALUE);
        mPointsAnimator.setRepeatMode(ValueAnimator.RESTART);
        mPointsAnimator.setRepeatCount(ValueAnimator.INFINITE);
        mPointsAnimator.addUpdateListener(animation -> {
            for (AnimPoint point : mPointList) {
                //通过属性动画不停的计算下粒子的下一个坐标
                point.updatePoint(mRandom, mRadius);
            }
            invalidate();
        });
        mPointsAnimator.start();
    }


    @Override
    protected void onDraw(final Canvas canvas) {
        super.onDraw(canvas);
        canvas.save();
        canvas.translate(mCenterX, mCenterY);
        //画运动粒子
        for (AnimPoint animPoint : mPointList) {
            mPointPaint.setAlpha(animPoint.getAlpha());
            canvas.drawCircle(animPoint.getmX(), animPoint.getmY(),
                    animPoint.getRadius(), mPointPaint);
        }
     }
```

#### 2.2、实现渐变色圆
实现圆从内到外渐变使用RadialGradient
大致实现方式如下
```java
float[] mRadialGradientStops = {0F, 0.69F, 0.86F, 0.94F, 0.98F, 1F};
mRadialGradientColors[0] = transparentColor;
mRadialGradientColors[1] = transparentColor;
mRadialGradientColors[2] = parameter.getInsideColor();
mRadialGradientColors[3] = parameter.getOutsizeColor();
mRadialGradientColors[4] = transparentColor;
mRadialGradientColors[5] = transparentColor;
mRadialGradient = new RadialGradient(
                    0,
                    0,
                    mCenterX,
                    mRadialGradientColors,
                    mRadialGradientStops,
                    Shader.TileMode.CLAMP);
mSweptPaint.setShader(mRadialGradient);

...
//onDraw()绘制
canvas.drawCircle(0, 0, mCenterX, mSweptPaint);
```
#### 2.3、展示背景圆的扇形区域
原本想通过DrawArc实现这个效果，但是DrawArc无法实现到圆心的区域
那么如何实现这么一个不规则的形状呢，可以使用canvas.clipPath()实现裁剪不规则的形状，所以只要得到扇形的Path就能实现，通过圆点+弧形再闭合path就能实现
![获取弧形path](https://upload-images.jianshu.io/upload_images/7312294-922e29e5a760f2e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```java
/**
     * 绘制扇形path
     *
     * @param r 半径
     * @param startAngle 开始角度
     * @param sweepAngle 扫过的角度
     */
private void getSectorClip(float r, float startAngle, float sweepAngle) {
        mArcPath.reset();
        mArcPath.addArc(-r, -r, r, r, startAngle, sweepAngle);
        mArcPath.lineTo(0, 0);
        mArcPath.close();
    }

//然后再onDraw()中，裁剪画布
 canvas.clipPath(mArcPath);
```
#### 2.4、实现指针变色
指针是不规则形状，无法通过绘制几何图形实现，所以选用drawBitmap实现
至于如何实现bitmap指针图片的颜色变化呢，原本的方案是使用AvoidXfermode改变指定像素通道范围内的颜色，但是AvoidXfermode在API 24已经被移除，所以这方案无效
最终采用图层混合模式实现指针图片变色
![16种图层混合模式](https://upload-images.jianshu.io/upload_images/7312294-6fc17b4997a56812.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过PorterDuff.Mode.MULTIPLY模式可以实现bitmap颜色，源图像为要修改的指针颜色，目标图像为白色指针，通过获取两个图像的重叠部分实现变色
大致实现如下
```java
/**
     * 初始化指针图片的Bitmap
     */
    private void initBitmap() {
        float f = 130F / 656F;
        mBitmapDST = BitmapFactory.decodeResource(getResources(), R.drawable.indicator);
        float mBitmapDstHeight = width * f;
        float mBitmapDstWidth = mBitmapDstHeight * mBitmapDST.getWidth() / mBitmapDST.getHeight();
        //初始化指针的图层混合模式
        mXfermode = new PorterDuffXfermode(PorterDuff.Mode.MULTIPLY);
        mPointerRectF = new RectF(0, 0, mBitmapDstWidth, mBitmapDstHeight);
        mBitmapSRT = Bitmap.createBitmap((int) mBitmapDstWidth, (int) mBitmapDstHeight, Bitmap.Config.ARGB_8888);
        mBitmapSRT.eraseColor(mIndicatorColor);
    }

    @Override
    protected void onDraw(final Canvas canvas) {
        super.onDraw(canvas);
        //画指针
       canvas.translate(mCenterX, mCenterY);
       canvas.rotate(mCurrentAngle / 10F);
       canvas.translate(-mPointerRectF.width() / 2, -mCenterY);
       mPointerLayoutId = canvas.saveLayer(mPointerRectF, mBmpPaint);
       mBitmapSRT.eraseColor(mIndicatorColor);
       canvas.drawBitmap(mBitmapDST, null, mPointerRectF, mBmpPaint);
       mBmpPaint.setXfermode(mXfermode);
       canvas.drawBitmap(mBitmapSRT, null, mPointerRectF, mBmpPaint);
       mBmpPaint.setXfermode(null);
       canvas.restoreToCount(mPointerLayoutId);
    }
```
#### 2.5、实现背景圆颜色随扇形角度变化
把圆形控件拆成3600°，每一个角度对应控件一种具体颜色值，那么如何计算特定角度他具体的颜色值呢？
参考属性动画中的变色动画android.animation.ArgbEvaluator实现方式，计算两个颜色中具体某一个点的颜色值方式如下
   ```java
public Object evaluate(float fraction, Object startValue, Object endValue) {
        int startInt = (Integer) startValue;
        float startA = ((startInt >> 24) & 0xff) / 255.0f;
        float startR = ((startInt >> 16) & 0xff) / 255.0f;
        float startG = ((startInt >>  8) & 0xff) / 255.0f;
        float startB = ( startInt        & 0xff) / 255.0f;

        int endInt = (Integer) endValue;
        float endA = ((endInt >> 24) & 0xff) / 255.0f;
        float endR = ((endInt >> 16) & 0xff) / 255.0f;
        float endG = ((endInt >>  8) & 0xff) / 255.0f;
        float endB = ( endInt        & 0xff) / 255.0f;

        // convert from sRGB to linear
        startR = (float) Math.pow(startR, 2.2);
        startG = (float) Math.pow(startG, 2.2);
        startB = (float) Math.pow(startB, 2.2);

        endR = (float) Math.pow(endR, 2.2);
        endG = (float) Math.pow(endG, 2.2);
        endB = (float) Math.pow(endB, 2.2);

        // compute the interpolated color in linear space
        float a = startA + fraction * (endA - startA);
        float r = startR + fraction * (endR - startR);
        float g = startG + fraction * (endG - startG);
        float b = startB + fraction * (endB - startB);

        // convert back to sRGB in the [0..255] range
        a = a * 255.0f;
        r = (float) Math.pow(r, 1.0 / 2.2) * 255.0f;
        g = (float) Math.pow(g, 1.0 / 2.2) * 255.0f;
        b = (float) Math.pow(b, 1.0 / 2.2) * 255.0f;

        return Math.round(a) << 24 | Math.round(r) << 16 | Math.round(g) << 8 | Math.round(b);
    }

```
控件中总共有四个颜色段，3600/4=900，所以 fraction = progressValue % 900 / 900;
然后判断当前的角度位于第几段颜色值中，通过android.animation.ArgbEvaluator.evaluate(float fraction, Object startValue, Object endValue) 就能回去具体的颜色值
大致实现过程如下
```java
private ProgressParameter getProgressParameter(float progressValue) {
        float fraction = progressValue % 900 / 900;
        if (progressValue < 900) {
            //第一个颜色段
            mParameter.setInsideColor(evaluate(fraction, insideColor1, insideColor2));
            mParameter.setOutsizeColor(evaluate(fraction, outsizeColor1, outsizeColor2));
            mParameter.setProgressColor(evaluate(fraction, progressColor1, progressColor2));
            mParameter.setPointColor(evaluate(fraction, pointColor1, pointColor2));
            mParameter.setBgCircleColor(evaluate(fraction, bgCircleColor1, bgCircleColor2));
            mParameter.setIndicatorColor(evaluate(fraction, indicatorColor1, indicatorColor2));
        } else if (progressValue < 1800) {
            //第二个颜色段
            mParameter.setInsideColor(evaluate(fraction, insideColor2, insideColor3));
            mParameter.setOutsizeColor(evaluate(fraction, outsizeColor2, outsizeColor3));
            mParameter.setProgressColor(evaluate(fraction, progressColor2, progressColor3));
            mParameter.setPointColor(evaluate(fraction, pointColor2, pointColor3));
            mParameter.setBgCircleColor(evaluate(fraction, bgCircleColor2, bgCircleColor3));
            mParameter.setIndicatorColor(evaluate(fraction, indicatorColor2, indicatorColor3));
        } else if (progressValue < 2700) {
            //第三个颜色段
            mParameter.setInsideColor(evaluate(fraction, insideColor3, insideColor4));
            mParameter.setOutsizeColor(evaluate(fraction, outsizeColor3, outsizeColor4));
            mParameter.setProgressColor(evaluate(fraction, progressColor3, progressColor4));
            mParameter.setPointColor(evaluate(fraction, pointColor3, pointColor4));
            mParameter.setBgCircleColor(evaluate(fraction, bgCircleColor3, bgCircleColor4));
            mParameter.setIndicatorColor(evaluate(fraction, indicatorColor3, indicatorColor4));
        } else {
            //第四个颜色段
            mParameter.setInsideColor(evaluate(fraction, insideColor4, insideColor5));
            mParameter.setOutsizeColor(evaluate(fraction, outsizeColor4, outsizeColor5));
            mParameter.setProgressColor(evaluate(fraction, progressColor4, progressColor5));
            mParameter.setPointColor(evaluate(fraction, pointColor4, pointColor5));
            mParameter.setBgCircleColor(evaluate(fraction, bgCircleColor4, bgCircleColor5));
            mParameter.setIndicatorColor(evaluate(fraction, indicatorColor4, indicatorColor5));
        }
        return mParameter;
    }

```

## 三、跳动数字动画实现

#### 3.1、属性动画+2个TextView实现数字上下切换动画
实现数字切换动画，原本打算用RecycleView实现，但是考虑到动效上将来可能面临UI小姐姐各种骚操作，所以最终决定就用两个TextView做上下translation动画，这样可控性高，对View执行属性动画也简单

NumberView使用FrameLayout包裹两个TextView，widget_progress_number_item_layout.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content">

    <TextView
        android:id="@+id/tv_number_one"
        style="@style/progress_text_font"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:gravity="center"
        android:padding="0dp"
        android:text="0"
        android:textColor="@android:color/white" />

    <TextView
        style="@style/progress_text_font"
        android:id="@+id/tv_number_tow"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:gravity="center"
        android:text="1"
        android:textColor="@android:color/white" />
</FrameLayout>
```
然后通过属性动画控制两个TextView上下切换
```java
mNumberAnim = ValueAnimator.ofFloat(0F, 1F);
        mNumberAnim.setDuration(400);
        mNumberAnim.setInterpolator(new OvershootInterpolator());
        mNumberAnim.setRepeatCount(0);
        mNumberAnim.setRepeatMode(ValueAnimator.RESTART);
        mNumberAnim.addUpdateListener(animation -> {
            float value = (float) animation.getAnimatedValue();
            if (UP_OR_DOWN_MODE == UP_ANIMATOR_MODE) {
                //数字变大，向下移动
                mTvFirst.setTranslationY(-mHeight * value);
                mTvSecond.setTranslationY(-mHeight * value);
            } else {
                //数字变小，向上移动
                mTvFirst.setTranslationY(mHeight * value);
                mTvSecond.setTranslationY(-2 * mHeight + mHeight * value);
            }
        });
```
这样NumberView就能实现一位数字的变化是上下切换动画，具有个十百位还有时钟冒号的通过容器布局AnimNumberView组合布局的方式实现表示时间和个十百位数

## 四、项目源码
博客只是大致讲了实现思路，具体实现请阅读源码
[https://github.com/DaLeiGe/AndroidSamples/tree/master/ProgressView](https://github.com/DaLeiGe/AndroidSamples/tree/master/ProgressView)

[本文为博主原创，转载请注明出处]([https://www.jianshu.com/p/c21fa3ecda25)


