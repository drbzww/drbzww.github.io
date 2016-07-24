---
title: 自定义任意形状的水波纹加载View
date: 2016-07-14 11:37:50
tags:
---
# 概述
对于一些耗时操作，例如加载大文件，如果不给用户一个动态加载的友好提示界面，就会让用户感觉应用卡死了，所以今天就来实现一个动态加载的自定义View(WaveProgressView)，先看一下实现后的效果图：
![最后效果图](http://upload-images.jianshu.io/upload_images/2171639-0edbab323a3e0bc8.gif?imageMogr2/auto-orient/strip)

# 实现代码详解
1. 通过自定义属性设置水波颜色、波长、振幅、波的颜色、字体大小、字体颜色、进度条最大值、当前进度值、波动的快慢和波动的方向，也可以通过set方法来实现。
自定义属性：
		<declare-styleable name="WaveProgressView">
			<attr name="wpv_max_progress" format="integer" />
			<attr name="wpv_current_progress" format="integer" />
			<attr name="wpv_progress_unit" format="reference|string" />
			<attr name="wpv_text_color" format="reference|color" />
			<attr name="wpv_text_size" format="reference|dimension" />
			<attr name="wpv_wave_color" format="reference|color" />
			<attr name="wpv_wave_width" format="reference|dimension" />
			<attr name="wpv_wave_height" format="reference|dimension" />
			<attr name="wpv_fluctuation_speed" >
				<enum name="slow" value="300" />
				<enum name="normal" value="200" />
				<enum name="fast" value="100" />
			</attr>
			<attr name="wpv_fluctuation_direction" >
				<enum name="left" value="0" />
				<enum name="right" value="1" />
			</attr>
		</declare-styleable>
set方法：
		public void setMaxProgress(int maxProgress) {
			this.mMaxProgress = maxProgress;
		}

		public void setmCurrentProgress(int currentProgress) {
			this.mCurrentProgress = currentProgress;
		}

		public void setProgressUnit(String progressUnit) {
			this.mProgressUnit = progressUnit;
		}

		public void setWaveColor(int waveColor) {
			this.mWaveColor = waveColor;
		}

		public void setTextColor(int textColor) {
			this.mTextColor = textColor;
		}

2. 获取背景图片然后将背景置为null
		if (null != getBackground()) {
			mBackGroundBmp = getBitmapFromDrawable(getBackground());
			setBackgroundDrawable(null);
		} else {
			throw new IllegalArgumentException(String.format("background is null."));
		}
上面的代码中再获取背景图片后不设置背景为null，会导致背景重复带来的圆角矩形背景圆角处没有被波浪填满的现象。就可以利用PorterDuff.Mode.DST_ATOP（取上层非交集部分与下层交集部分）在后面会将背景图片和绘制的波浪重叠在一起

3. 利用二阶贝塞尔曲线绘制波浪
关于贝塞尔曲线可以参考[贝塞尔曲线扫盲](http://www.html-js.com/article/1628)
1> 首先算出在指定宽度的View中实现波动效果最少要绘制几个波长的贝塞尔曲线：
		mWaveNumber = (int) (mViewWidth / mWaveWidth + 2);
假如View的宽度是波长的2.2倍，那么在指定宽度的View中绘制一个波动的波浪最少需要绘制3.2个波长的贝塞尔曲线，为了方便我就绘制4个波长的贝塞尔曲线，也就是上面代码中加2的原因。
2> 根据波浪波动的方向得到绘制贝塞尔曲线的起始点x坐标，如果波浪向左移动的话，那么绘制贝塞尔曲线的起始点x坐标就是VIew左上角x坐标，如果波浪向右移动的话，那么绘制贝塞尔曲线的起始点x坐标就是VIew左上角向左移动一个波长距离后得到的点x坐标。
		switch (mFluctuationDirection) {
			case LEFT:
				offset = 0;
				steps = -TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, 20, getResources().getDisplayMetrics());
				break;
			case RIGHT:
				offset = -mWaveWidth;
				steps = TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, 20, getResources().getDisplayMetrics());;
				break;
		}
上面代码中就是对offset初始化（即绘制贝塞尔曲线的起始点x坐标） ，steps 是每一次在水平方向移动的距离。
3> 利用Path绘制效果图中的绿色波浪
		Path path = new Path();
		path.moveTo(offset, mCurrentWaveY);
		for (int i = 0; i < mWaveNumber; i++) {
			path.quadTo(mWaveWidth * i + offset + mWaveWidth / 4, mCurrentWaveY - mWaveHeight, mWaveWidth * i + offset + mWaveWidth / 2, mCurrentWaveY);
			path.quadTo(mWaveWidth * i + offset + mWaveWidth * 3 / 4, mCurrentWaveY + mWaveHeight, mWaveWidth * i + offset + mWaveWidth, mCurrentWaveY);
		}
		path.lineTo(mViewWidth, mViewHeight);
		path.lineTo(0,mViewHeight);
		path.close();
		canvas.drawPath(path, mWavePaint);
![波形图](http://upload-images.jianshu.io/upload_images/2171639-64dbb2ca9b6e39b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我就是按照上图的波形来绘制贝塞尔曲线的，上面代码中的mCurrentWaveY代表波浪中线(相当于上图中的x轴)的y轴坐标；首先设置绘制贝塞尔曲线的起始点为上图中与x轴最左边的交点；然后通过for循环绘制mWaveNumber个波长的贝塞尔曲线，每个波长的贝塞尔曲线有两条二阶贝塞尔曲线组成，第一条代表一个波长的上半部分，第二条代表一个波长的下半部分；绘制完贝塞尔曲线后就是依次连接View的右下角坐标和左下角坐标，从而完成绿色波浪的绘制。
4. 利用PorterDuff.Mode.DST_ATOP(取上层非交集部分与下层交集部分)将背景图片绘制在绿色波浪上面，从而可以得到任意形状的波浪；接着再把描述进度的文字绘制上去：
		Paint paint = new Paint();
		paint.setAntiAlias(true);
		paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_ATOP));
		canvas.drawBitmap(Bitmap.createScaledBitmap(mBackGroundBmp, mViewWidth, mViewHeight, false), 0, 0, paint);
		canvas.drawText(mCurrentProgress + mProgressUnit, mViewWidth / 2,
				mViewHeight / 2 - (mTextPaint.getFontMetricsInt().ascent + mTextPaint.getFontMetricsInt().descent) / 2, mTextPaint);
上面代码中最后面利用drawText居中绘制字符串，关于如何居中绘制字符串请参考我的另一篇博客：
[在Canvas中居中显示文字](http://www.jianshu.com/p/b3305a082034)

5. 绘制波浪的完整代码如下：
		@Override
		protected void onDraw(Canvas canvas) {
			//super.onDraw(canvas);
			canvas.drawBitmap(createImage(), 0, 0, null);
		}
		private float mCurrentWaveY;
		private int mWaveNumber;
		private Bitmap createImage()
		{
			Bitmap finalBmp = Bitmap.createBitmap(mViewWidth, mViewHeight, Bitmap.Config.ARGB_8888);
			Canvas canvas = new Canvas(finalBmp);
			Path path = new Path();
			path.moveTo(offset, mCurrentWaveY);
			for (int i = 0; i < mWaveNumber; i++) {
				path.quadTo(mWaveWidth * i + offset + mWaveWidth / 4, mCurrentWaveY - mWaveHeight, mWaveWidth * i + offset + mWaveWidth / 2, mCurrentWaveY);
				path.quadTo(mWaveWidth * i + offset + mWaveWidth * 3 / 4, mCurrentWaveY + mWaveHeight, mWaveWidth * i + offset + mWaveWidth, mCurrentWaveY);
			}
			path.lineTo(mViewWidth, mViewHeight);
			path.lineTo(0,mViewHeight);
			path.close();
			canvas.drawPath(path, mWavePaint);

			Paint paint = new Paint();
			paint.setAntiAlias(true);
			paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_ATOP));
			canvas.drawBitmap(Bitmap.createScaledBitmap(mBackGroundBmp, mViewWidth, mViewHeight, false), 0, 0, paint);

			canvas.drawText(mCurrentProgress + mProgressUnit, mViewWidth / 2,
					mViewHeight / 2 - (mTextPaint.getFontMetricsInt().ascent + mTextPaint.getFontMetricsInt().descent) / 2, mTextPaint);

			return finalBmp;
		}
6. 让波浪波动起来
		private Handler mHandler = new Handler() {
			@Override
			public void handleMessage(Message msg) {
				super.handleMessage(msg);
				waveMoveToNext();
				invalidate();
				mHandler.sendEmptyMessageDelayed(0, mFluctuationSpeed.getValue());
			}
		};
		private void waveMoveToNext() {
			offset += steps;
			if (offset < -mWaveWidth) {
				offset += mWaveWidth;
			} else if (offset > 0) {
				offset -= mWaveWidth;
			}

			if (refreshRate > 0) {
				mCurrentWaveY -= (1.0f / mMaxProgress) * mViewHeight / 5;
				refreshRate--;
			} else {
				mCurrentProgress ++;
				if (mCurrentProgress > 100) {
					mCurrentProgress = 0;
					mCurrentWaveY = mViewHeight;
				}
				refreshRate = 5;
			}
		}
在构造方法中发送一个消息给Hander即可实现波动，另外为了每次波浪上升的时候不产生卡顿效果，把这1/mMaxProgress的上升分为5次来绘制。