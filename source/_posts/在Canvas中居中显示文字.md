---
title: 在Canvas中居中显示文字
date: 2016-07-12 18:38:18
tags:
---

我们在自定义View中有的时候会想自己绘制文字，自己绘制文字的时候，我们通常希望把文字精确定位，文字居中（水平、垂直）是普遍的需求，所以这里就以文字居中为例。Android是通过Canvas中的drawText方法进行文字绘制的，方法使用说明如下：
```
public void drawText(@NonNull String text, int start, int end, float x, float y, @NonNull Paint paint) {
text：要绘制的字符串
start：第一个要绘制字符的下标值
end：最后一个要绘制字符的下标值
x默认是字符串的左边在屏幕的位置，如果设置了paint.setTextAlign(Paint.Align.CENTER);那就是字符串的中心对应的x坐标，y是指定字符串baseline在屏幕上的位置。
```

Canvas绘制文本时，通过Paint对象获取FontMetrics对象，然后利用FontMetrics对象计算baseline在屏幕上的位置。 它的思路和java.awt.FontMetrics的基本相同。 FontMetrics对象它以四个基本坐标为基准，如下图所示：

![基准线视图](http://upload-images.jianshu.io/upload_images/2171639-d83c9ccd6d7a57ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
FontMetrics.top       该距离是从所绘字符的baseline之上至可绘制区域的最高点。
FontMetrics.ascent    该距离是从所绘字符的baseline之上至该字符所绘制的最高点。这个距离是系统推荐。
FontMetrics.descent   该距离是从所绘字符的baseline之下至该字符所绘制的最低点。这个距离是系统推荐的。
FontMetrics.bottom    该距离是从所绘字符的baseline之下至可绘制区域的最低点。
```
由上图可以知道字符串的绘制区域在FontMetrics.ascent和FontMetrics.descent之间，因此让字符串垂直居中显示就相当于让字符串的绘制区域垂直居中显示；由于drawText方法中y参数所需要的值就是图中的红线（baseline）对应的y值，因此只要计算出字符串的绘制区域垂直居中显示时红线（baseline）对应的y值即可，计算过程如下：
```
假设我们所求的baseline的值为baseY;
text的descent距离：
①descentY = baseY + fontMetrics.descent; 
text的字体高度：
②fontHeight = fontMetrics.descent- fontMetrics.ascent 
因为我们要让text垂直居中，所以此时text的bottom距离应该为：
③descentY=1/2 * height + 1/2 * fontHeight
所以由上述①②③公式就可以推得：
baseY = 1/2 * height － 1/2 * (fontMetrics.ascent ＋ fontMetrics.descent） 
此时求得baseline的值，即cavans.drawText()里的y的坐标。
```

获取fontMetrics的方法如下：
```
mFontMetricsInt = mPaint.getFontMetricsInt();
```