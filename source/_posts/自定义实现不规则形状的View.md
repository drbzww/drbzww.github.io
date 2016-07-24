---
title: 自定义实现不规则形状的View
date: 2016-07-06 14:05:11
tags:
---

# 概述
在一些项目中要求将头像显示成圆形或者其他的一些不规则形状的图形，我们不可能为了实现这样的效果，在代码中将图像进行裁剪，这样的话也显得太low了，也没有扩展性。一般实现自定义形状的图形有三种方式：PorterDuffXfermode 、BitmapShader、ClipPath。下面我都会分别说明，我这里的实现使用的是第一种方式，实现效果图如下所示：

![运行效果图](http://upload-images.jianshu.io/upload_images/2171639-59cf0838a21660bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# PorterDuffXfermode 方式
这是由Tomas Proter和 Tom Duff命名的图像转换模式，它有16个枚举值来控制Canvas上 上下两个图层的交互（先画的图层在下层）。

![蓝色的在上层](http://upload-images.jianshu.io/upload_images/2171639-da149a0c83d741c6?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

     1.PorterDuff.Mode.CLEAR　　 　所绘制不会提交到画布上 
     2.PorterDuff.Mode.SRC　　　　　显示上层绘制图片 
     3.PorterDuff.Mode.DST　　　　　显示下层绘制图片 
     4.PorterDuff.Mode.SRC_OVER　　正常绘制显示，上下层绘制叠盖。 
     5.PorterDuff.Mode.DST_OVER　　上下层都显示。下层居上显示。 
     6.PorterDuff.Mode.SRC_IN　　　　取两层绘制交集。显示上层。 
     7.PorterDuff.Mode.DST_IN　　　　取两层绘制交集。显示下层。 
     8.PorterDuff.Mode.SRC_OUT　　　取上层绘制非交集部分。 
     9.PorterDuff.Mode.DST_OUT　　　取下层绘制非交集部分。 
     10.PorterDuff.Mode.SRC_ATOP　　取下层非交集部分与上层交集部分 
     11.PorterDuff.Mode.DST_ATOP　　取上层非交集部分与下层交集部分 
     12.PorterDuff.Mode.XOR　　　　　异或：去除两图层交集部分 
     13.PorterDuff.Mode.DARKEN　　　取两图层全部区域，交集部分颜色加深 
     14.PorterDuff.Mode.LIGHTEN　　　取两图层全部，点亮交集部分颜色 
     15.PorterDuff.Mode.MULTIPLY　　取两图层交集部分叠加后颜色 
     16.PorterDuff.Mode.SCREEN　　　 取两图层全部区域，交集部分变为透明色

1. 实现思路
会玩Ps的朋友肯定知道，如果有两个图层，我们想把上面图层裁切成下面图层的形状，只需要调下面图层的选区，然后选中上面的图层，蒙板就可以了。那么我们就可以利用PorterDuff.Mode的 SRC_IN 或 DST_IN 来取得两个图层的交集，从而把图像裁切成我们想要的各种样式。我们需要一个形状图层和一个显示图层。并且显示图层要完全覆盖形状图层。

2. 代码实现与详解
继承ImageView，复写了imageview的四个setImage方法（为了更好的兼容性），在setImageDrawable方法中得到前景图片（即显示图层）：

```
@Override
public void setImageBitmap(Bitmap bm) {
    Log.d(TAG, "setImageBitmap bm = " + bm);
    super.setImageBitmap(bm);
    mBitmap = bm;
    setBackgroundBmp();
}

@Override
public void setImageDrawable(Drawable drawable) {
    Log.d(TAG, "setImageDrawable drawable = " + drawable);
    super.setImageDrawable(drawable);
    mBitmap = getBitmapFromDrawable(drawable);
    setBackgroundBmp();
}

@Override
public void setImageResource(int resId) {
    Log.d(TAG, "setImageResource resId = " + resId);
    super.setImageResource(resId);
    mBitmap = getBitmapFromDrawable(getDrawable());
    setBackgroundBmp();
}

@Override
public void setImageURI(Uri uri) {
    super.setImageURI(uri);
    Log.d(TAG, "setImageURI uri = " + uri);
    mBitmap = getBitmapFromDrawable(getDrawable());
    setBackgroundBmp();
}
```

然后获取背景图片（即形状图层）并且通过调用invalidate()使View被重绘：
```
private void setBackgroundBmp(){
    if(null==getBackground()){
        throw new IllegalArgumentException(String.format("background is null."));
    }else{
        mBackgroundBmp = getBitmapFromDrawable(getBackground());
        invalidate();
    }
}
```
在重绘之前利用PorterDuff.Mode的 SRC_IN 或 DST_IN 取形状图层和显示图层交集，从而得到自定义形状的图片：
```
@Override
protected void onSizeChanged(int w, int h, int oldw, int oldh) {
    super.onSizeChanged(w, h, oldw, oldh);
    mViewWidth = w;
    mViewHeight = h;
}

private Bitmap createImage()
{
    mBackgroundBmp = getCenterCropBitmap(mBackgroundBmp, mViewWidth, mViewHeight);
    mBitmap = getCenterCropBitmap(mBitmap, mViewWidth, mViewHeight);

    int bmpWidth = mBitmap.getWidth();
    int bmpHeight = mBitmap.getHeight();

    Paint paint = new Paint();
    paint.setAntiAlias(true);
    Bitmap finalBmp = Bitmap.createBitmap(mViewWidth,mViewHeight, Bitmap.Config.ARGB_8888);
    Canvas canvas = new Canvas(finalBmp);
    canvas.drawBitmap(mBackgroundBmp, 0, 0, paint);
    paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_IN));
    canvas.drawBitmap(mBitmap, (mViewWidth - bmpWidth) / 2, (mViewHeight - bmpHeight) / 2, paint);
    return finalBmp;
}

/**
 * 类比ScaleType.CENTER_CROP
 */
private Bitmap getCenterCropBitmap(Bitmap src, float rectWidth, float rectHeight) {

    float srcRatio = ((float) src.getWidth()) / src.getHeight();
    float rectRadio = rectWidth / rectHeight;
    if (srcRatio < rectRadio) {
        return Bitmap.createScaledBitmap(src, (int)rectWidth, (int)((rectWidth / src.getWidth()) * src.getHeight()), false);
    } else {
        return Bitmap.createScaledBitmap(src, (int)((rectHeight / src.getHeight()) * src.getWidth()), (int)rectHeight, false);
    }
}
```

得到要绘制的图片后，就是通过重写onDraw方法进行绘制：
```
@Override
protected void onDraw(Canvas canvas) {
    if(mBitmap!=null && mBackgroundBmp!=null){
        canvas.drawBitmap(createImage(), 0, 0, null);
    }
}
```
然后就是在布局中使用了：
```
<com.cytmxk.customview.customshapeview.AvatarView
    android:layout_marginLeft="20dp"
    android:layout_width="100dp"
    android:layout_height="100dp"
    android:background="@drawable/avatar_view_rectangle_shape_two"
    android:src="@drawable/avatar_view2" />
```
这里的android:background定义的就是我们的形状图层，它可以是一个xxx_shape.xml的布局文件，比如：
```
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle" >
    <solid android:color="#fff" ></solid>
    <corners android:radius="10dp" />
    <size android:width="100dp"
        android:height="100dp"/>
</shape>
```

# BitmapShader方式
通过Bitmap得到一个着色器：
```
mBitmapShader = new BitmapShader(mBitmap, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
参数 
①　mBitmap：你要绘制的图片
②　emun Shader.TileMode 定义了三种着色模式： 
　　 CLAMP 拉伸
 　　REPEAT 重复
 　　MIRROR 镜像
 好比你拿一张分辨率和电脑屏幕不一样的图片设置为壁纸时，选择的三种方式一样。
```
给画笔设置着色器，这样画笔就能在 canvas的相应形状上画出我们的图片Bitmap:
```
mBitmapPaint.setShader(mBitmapShader);
canvas.drawCircle(getWidth() / 2, getHeight() / 2, mDrawableRadius, mBitmapPaint);
```
当然我们一般设置的模式为CLAMP 拉伸（当图片mBitmap的宽高小于View的时候要拉伸），但是我们一般不想要拉伸（变形了），所以一般还要给着色器设置一个matrix，去适当的放大或者缩小图片。
```
mBitmapShader.setLocalMatrix(mShaderMatrix);
```
著名的项目[CircleImageView](https://github.com/hdodenhof/CircleImageView)就是用着色器实现的，实现思路上面已经说了，代码有详细的注释，理解起来应该没什么问题：
```
public class CircleImageView extends ImageView {
    //缩放类型
    private static final ScaleType SCALE_TYPE = ScaleType.CENTER_CROP;
    private static final Bitmap.Config BITMAP_CONFIG = Bitmap.Config.ARGB_8888;
    private static final int COLORDRAWABLE_DIMENSION = 2;
    // 默认边界宽度
    private static final int DEFAULT_BORDER_WIDTH = 0;
    // 默认边界颜色
    private static final int DEFAULT_BORDER_COLOR = Color.BLACK;
    private static final boolean DEFAULT_BORDER_OVERLAY = false;

    private final RectF mDrawableRect = new RectF();
    private final RectF mBorderRect = new RectF();

    private final Matrix mShaderMatrix = new Matrix();
    //这个画笔最重要的是关联了mBitmapShader 使canvas在执行的时候可以切割原图片(mBitmapShader是关联了原图的bitmap的)
    private final Paint mBitmapPaint = new Paint();
    //这个描边，则与本身的原图bitmap没有任何关联，
    private final Paint mBorderPaint = new Paint();
    //这里定义了 圆形边缘的默认宽度和颜色
    private int mBorderColor = DEFAULT_BORDER_COLOR;
    private int mBorderWidth = DEFAULT_BORDER_WIDTH;

    private Bitmap mBitmap;
    private BitmapShader mBitmapShader; // 位图渲染
    private int mBitmapWidth;   // 位图宽度
    private int mBitmapHeight;  // 位图高度

    private float mDrawableRadius;// 图片半径
    private float mBorderRadius;// 带边框的的图片半径

    private ColorFilter mColorFilter;
    //初始false
    private boolean mReady;
    private boolean mSetupPending;
    private boolean mBorderOverlay;
    //构造函数
    public CircleImageView(Context context) {
        super(context);
        init();
    }
    //构造函数
    public CircleImageView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }
    /**
     * 构造函数
     */
    public CircleImageView(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.CircleImageView, defStyle, 0);
        //通过TypedArray提供的一系列方法getXXXX取得我们在xml里定义的参数值；
        // 获取边界的宽度
        mBorderWidth = a.getDimensionPixelSize(R.styleable.CircleImageView_border_width, DEFAULT_BORDER_WIDTH);
        // 获取边界的颜色
        mBorderColor = a.getColor(R.styleable.CircleImageView_border_color, DEFAULT_BORDER_COLOR);
        mBorderOverlay = a.getBoolean(R.styleable.CircleImageView_border_overlay, DEFAULT_BORDER_OVERLAY);
        //调用 recycle() 回收TypedArray,以便后面重用
        a.recycle();
        init();
    }
    /**
     * 作用就是保证第一次执行setup函数里下面代码要在构造函数执行完毕时调用
     */
    private void init() {
        //在这里ScaleType被强制设定为CENTER_CROP，就是将图片水平垂直居中，进行缩放。
        super.setScaleType(SCALE_TYPE);
        mReady = true;

        if (mSetupPending) {
            setup();
            mSetupPending = false;
        }
    }

    @Override
    public ScaleType getScaleType() {
        return SCALE_TYPE;
    }
    /**
     * 这里明确指出 此种imageview 只支持CENTER_CROP 这一种属性
     *
     * @param scaleType
     */
    @Override
    public void setScaleType(ScaleType scaleType) {
        if (scaleType != SCALE_TYPE) {
            throw new IllegalArgumentException(String.format("ScaleType %s not supported.", scaleType));
        }
    }

    @Override
    public void setAdjustViewBounds(boolean adjustViewBounds) {
        if (adjustViewBounds) {
            throw new IllegalArgumentException("adjustViewBounds not supported.");
        }
    }

    @Override
    protected void onDraw(Canvas canvas) {
        //如果图片不存在就不画
        if (getDrawable() == null) {
            return;
        }
        //绘制内圆形 图片 画笔为mBitmapPaint
        canvas.drawCircle(getWidth() / 2, getHeight() / 2, mDrawableRadius, mBitmapPaint);
        //如果圆形边缘的宽度不为0 我们还要绘制带边界的外圆形 边界画笔为mBorderPaint
        if (mBorderWidth != 0) {
            canvas.drawCircle(getWidth() / 2, getHeight() / 2, mBorderRadius, mBorderPaint);
        }
    }

    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
        setup();
    }

    public int getBorderColor() {
        return mBorderColor;
    }

    public void setBorderColor(int borderColor) {
        if (borderColor == mBorderColor) {
            return;
        }

        mBorderColor = borderColor;
        mBorderPaint.setColor(mBorderColor);
        invalidate();
    }

    public void setBorderColorResource( int borderColorRes) {
        setBorderColor(getContext().getResources().getColor(borderColorRes));
    }

    public int getBorderWidth() {
        return mBorderWidth;
    }

    public void setBorderWidth(int borderWidth) {
        if (borderWidth == mBorderWidth) {
            return;
        }

        mBorderWidth = borderWidth;
        setup();
    }

    public boolean isBorderOverlay() {
        return mBorderOverlay;
    }

    public void setBorderOverlay(boolean borderOverlay) {
        if (borderOverlay == mBorderOverlay) {
            return;
        }

        mBorderOverlay = borderOverlay;
        setup();
    }

    /**
     * 以下四个函数都是
     * 复写ImageView的setImageXxx()方法
     * 注意这个函数先于构造函数调用之前调用
     * @param bm
     */
    @Override
    public void setImageBitmap(Bitmap bm) {
        super.setImageBitmap(bm);
        mBitmap = bm;
        setup();
    }

    @Override
    public void setImageDrawable(Drawable drawable) {
        super.setImageDrawable(drawable);
        mBitmap = getBitmapFromDrawable(drawable);
        setup();
    }

    @Override
    public void setImageResource( int resId) {
        super.setImageResource(resId);
        mBitmap = getBitmapFromDrawable(getDrawable());
        setup();
    }

    @Override
    public void setImageURI(Uri uri) {
        super.setImageURI(uri);
        mBitmap = getBitmapFromDrawable(getDrawable());
        setup();
    }

    @Override
    public void setColorFilter(ColorFilter cf) {
        if (cf == mColorFilter) {
            return;
        }

        mColorFilter = cf;
        mBitmapPaint.setColorFilter(mColorFilter);
        invalidate();
    }
    /**
     * Drawable转Bitmap
     * @param drawable
     * @return
     */
    private Bitmap getBitmapFromDrawable(Drawable drawable) {
        if (drawable == null) {
            return null;
        }

        if (drawable instanceof BitmapDrawable) {
            //通常来说 我们的代码就是执行到这里就返回了。返回的就是我们最原始的bitmap
            return ((BitmapDrawable) drawable).getBitmap();
        }

        try {
            Bitmap bitmap;

            if (drawable instanceof ColorDrawable) {
                bitmap = Bitmap.createBitmap(COLORDRAWABLE_DIMENSION, COLORDRAWABLE_DIMENSION, BITMAP_CONFIG);
            } else {
                bitmap = Bitmap.createBitmap(drawable.getIntrinsicWidth(), drawable.getIntrinsicHeight(), BITMAP_CONFIG);
            }

            Canvas canvas = new Canvas(bitmap);
            drawable.setBounds(0, 0, canvas.getWidth(), canvas.getHeight());
            drawable.draw(canvas);
            return bitmap;
        } catch (OutOfMemoryError e) {
            return null;
        }
    }
    /**
     * 这个函数很关键，进行图片画笔边界画笔(Paint)一些重绘参数初始化：
     * 构建渲染器BitmapShader用Bitmap来填充绘制区域,设置样式以及内外圆半径计算等，
     * 以及调用updateShaderMatrix()函数和 invalidate()函数；
     */
    private void setup() {
        //因为mReady默认值为false,所以第一次进这个函数的时候if语句为真进入括号体内
        //设置mSetupPending为true然后直接返回，后面的代码并没有执行。
        if (!mReady) {
            mSetupPending = true;
            return;
        }
        //防止空指针异常
        if (mBitmap == null) {
            return;
        }
        // 构建渲染器，用mBitmap位图来填充绘制区域,参数值代表如果图片太小的话 就直接拉伸
        mBitmapShader = new BitmapShader(mBitmap, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
        // 设置图片画笔反锯齿
        mBitmapPaint.setAntiAlias(true);
        // 设置图片画笔渲染器
        mBitmapPaint.setShader(mBitmapShader);
        // 设置边界画笔样式
        mBorderPaint.setStyle(Paint.Style.STROKE);//设画笔为空心
        mBorderPaint.setAntiAlias(true);
        mBorderPaint.setColor(mBorderColor);    //画笔颜色
        mBorderPaint.setStrokeWidth(mBorderWidth);//画笔边界宽度
        //这个地方是取的原图片的宽高
        mBitmapHeight = mBitmap.getHeight();
        mBitmapWidth = mBitmap.getWidth();
        // 设置含边界显示区域，取的是CircleImageView的布局实际大小，为方形
        mBorderRect.set(0, 0, getWidth(), getHeight());
        //计算 圆形带边界部分（外圆）的最小半径，取mBorderRect的宽高减去一个边缘大小的一半的较小值
        mBorderRadius = Math.min((mBorderRect.height() - mBorderWidth) / 2, (mBorderRect.width() - mBorderWidth) / 2);
        // 初始图片显示区域为mBorderRect（CircleImageView的布局实际大小）
        mDrawableRect.set(mBorderRect);
        if (!mBorderOverlay) {
            //demo里始终执行
            //通过inset方法  使得图片显示的区域从mBorderRect大小上下左右内移边界的宽度形成区域
            mDrawableRect.inset(mBorderWidth, mBorderWidth);
        }
        //这里计算的是内圆的最小半径，也即去除边界宽度的半径
        mDrawableRadius = Math.min(mDrawableRect.height() / 2, mDrawableRect.width() / 2);
        //设置渲染器的变换矩阵也即是mBitmap用何种缩放形式填充
        updateShaderMatrix();
        //手动触发ondraw()函数 完成最终的绘制
        invalidate();
    }
    /**
     * 这个函数为设置BitmapShader的Matrix参数，设置最小缩放比例，平移参数。
     * 作用：保证图片损失度最小和始终绘制图片正中央的那部分
     */
    private void updateShaderMatrix() {
        float scale;
        float dx = 0;
        float dy = 0;

        mShaderMatrix.set(null);
        // 这里不好理解 这个不等式也就是(mBitmapWidth / mDrawableRect.width()) > (mBitmapHeight / mDrawableRect.height())
        //取最小的缩放比例
        if (mBitmapWidth * mDrawableRect.height() > mDrawableRect.width() * mBitmapHeight) {
            //y轴缩放 x轴平移 使得图片的y轴方向的边的尺寸缩放到图片显示区域（mDrawableRect）一样）
            scale = mDrawableRect.height() / (float) mBitmapHeight;
            dx = (mDrawableRect.width() - mBitmapWidth * scale) * 0.5f;
        } else {
            //x轴缩放 y轴平移 使得图片的x轴方向的边的尺寸缩放到图片显示区域（mDrawableRect）一样）
            scale = mDrawableRect.width() / (float) mBitmapWidth;
            dy = (mDrawableRect.height() - mBitmapHeight * scale) * 0.5f;
        }
        // shaeder的变换矩阵，我们这里主要用于放大或者缩小。
        mShaderMatrix.setScale(scale, scale);
        // 平移
        mShaderMatrix.postTranslate((int) (dx + 0.5f) + mDrawableRect.left, (int) (dy + 0.5f) + mDrawableRect.top);
        // 设置变换矩阵
        mBitmapShader.setLocalMatrix(mShaderMatrix);
    }

}
```

# ClipPath方式
以下代码可以把图形bitmap画在一个圆上，得到一个圆形头像：
```
@Override
protected void onDraw(Canvas canvas) {
    Path path = new Path(); 
    //按照逆时针方向添加一个圆
    path.addCircle(float x, float y, mRadius, Direction.CCW);
    //先将canvas保存
    canvas.save();
    //把canvas修剪成指定的路径区域
    canvas.clipPath(path);
    //绘制图形Bitmap
    canvas.drawBitmap(Bitmap,float left, float top, mPaint);
    //恢复Canvas
    canvas.restore();
}
```
这种方式明显最简单，你还可以一个个坐标点的添加形成一个路径。但是形状比较复杂的情况下，还是第一种实现比较方便。