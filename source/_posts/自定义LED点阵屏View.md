---
title: 自定义LED点阵屏View
date: 2016-07-08 16:42:03
tags:
---

# 概述
看过演唱会的同学应该都看到过粉丝举着LED点阵屏幕的牌子来支持自己心目中的男神或者女神，感觉这种点阵屏幕的效果挺有意思的，于是花了点时间用Android实现了一下，实现效果如下：

![实现效果](http://upload-images.jianshu.io/upload_images/2171639-dbecaa5064916cad.gif?imageMogr2/auto-orient/strip)

# 相关的概念
1. 点阵字库与矢量字库
点阵字库就是每一个汉字用矩形点阵来表示，然后用每个点的虚实来表示汉字的轮廓，常用来作为显示字库使用，这类点阵字库汉字最大的缺点是不能放大，一旦放大后就会发现文字边缘的锯齿，常用的点阵矩阵有HZK12、HZK16和HZK24，我下面例子中的用到的字库是HZK16。
矢量字库保存的是对每一个汉字的描述信息，比如一个笔划的起始、终止坐标，半径、弧度等等。在显示、打印这一类字库时，要经过一系列的数学运算才能输出结果，但是这一类字库保存的汉字理论上可以被无限地放大，笔划轮廓仍然能保持圆滑，打印时使用的字库均为此类字库.

2. 点阵字库结构
在汉字的点阵字库中，每个字节的每个位都代表一个汉字的一个点，每个汉字都是由一个矩形点阵组成，0代表没有，1代表有点，将0和1分别用不同颜色画出，就形成了一个汉字。字库根据字节所表示的点是一行还是一列将字库的存储方式分为横向和纵向，目前多数的字库都是横向的存储方式(用得最多的应该是早期UCDOS字库)，纵向一般是因为有某些液晶是采用纵向扫描显示法，为了提高显示速度，于是便把字库矩阵做成纵向，省得在显示时还要做矩阵转换。我们接下去所描述的HZK16就是一种纵向字库。对于16\*16字库来说，它所需要的位数共是16\*16＝256个位，每个字节为8位，因此，每个汉字都需要用256/8=32个字节来表示。即每两个字节代表一行的16个点，共需要16行，显示汉字时，只需一次性读取32个字节，并将每两个字节为一行打印出来，即可形成一个汉字.

3. 汉字的区位码
汉字通过GB2312编码即每个汉字用两个byte来表示，第一个byte表示这个汉字在字库文件中的区码，第二个byte表示这个汉字在字库文件中的位码，通过这两个值可以计算到这个汉字在字库文件中的相对位置，根据这个位置读取接下来的32个byte(对于16\*16字库)，就对应着这个汉字对应的字模信息，字模信息其实就是一个byte数组。
**HZK16字库**是符合GB2312标准的16×16点阵字库，HZK16的GB2312-80支持的汉字有6763个，符号682个。其中一级汉字有3755个，按声序排列，二级汉字有3008个，按偏旁部首排列。

4. 通过机内码获取文字对应字模信息的起始位置
在PC机的文本文件中，汉字是以机内码的形式存储的，每个汉字占用两个字节：第一个字节为区码，为了与ASCII码区别，范围从十六进制的0A1H开始（小于80H的为ASCII码字符），对应区位码中区码的第一区；第二个字节为位码，范围也是从0A1H开始，对应某区中的第一个位码。这样，将汉字机内码减去0A0A0H就得该汉字的区位码。
例如汉字“房”的机内码为十六进制的“B7BF”，其中“B7”表示区码，“BF”表示位码。所以“房”的区位码为0B7BFH-0A0A0H=171FH。将区码和位码分别转换为十进制得汉字“房”的区位码为“2331”，即“房”的字模信息位于第23区的第31个字的位置，由于一个区包含94个汉字，所以第32×[(23-1) ×94+(31-1)]=67136Bit以后的32个字节为“房”的字模信息。

# 代码实现解析
要实现上面的滚动字幕的效果，可以分为如下几步：
1. 获取文字字符串的字模信息，并且将字模信息转化为Boolean类型的二维数组(字模信息的每一bit中0代表没有，1代表有点，将0转换成false，将1转换为true)
```
    /**
     * 获取汉字字符串的点阵矩阵
     * @param text
     * @return
     */
    public boolean[][] getWordsMatrix(Context context, String text) {
        return getWordsMatrix(context, text, null);
    }

    public boolean[][] getWordsMatrix(Context context, String text, DotMatrixFontType dotMatrixFontType) {
        if (null == dotMatrixFontType) {
            this.mDotMatrixFontType = DotMatrixFontType.SIXTEEN_TYPE;
            this.mWordByteByDots = DotMatrixFontType.SIXTEEN_TYPE.getValue() * DotMatrixFontType.SIXTEEN_TYPE.getValue() / 8;
        } else {
            this.mDotMatrixFontType = dotMatrixFontType;
            this.mWordByteByDots = dotMatrixFontType.getValue() * dotMatrixFontType.getValue() / 8;
        }

        byte[] bytes = null;
        try {
            // 获取汉字文本的字节编码
            bytes = text.getBytes(ENCODE);
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        // 获取每个字节对应正数编码，即得到汉字对应的区码和位码
        int[] code = new int[bytes.length];
        for (int i = 0; i < bytes.length; i++) {
            code[i] = bytes[i] < 0 ? 256 + bytes[i] : bytes[i];
        }

        int wordNumber = code.length / 2;
        boolean[][] wordsMatrix = new boolean[mDotMatrixFontType.getValue()][mDotMatrixFontType.getValue() * wordNumber];
        for (int i = 0; i < wordNumber; i++) {
            // 通过区码和位码获取字库中对应的字模信息
            byte[] temp = read(context, code[2 * i], code[2 * i + 1]);
            for (int j = 0; j < mWordByteByDots; j++) {
                for (int k = 0; k < 8; k++) {
                    // 将字模信息转化为Boolean类型的二维数组并且进行纵向填充数组
                    int row = (j * 8 + k) / 16 + i * mDotMatrixFontType.getValue();
                    int col = (j * 8 + k) % 16;
                    if (((temp[j] >> (7 - k)) & 1) == 1) {
                        wordsMatrix[col][row] = true;
                    } else {
                        wordsMatrix[col][row] = false;
                    }
                }
            }
        }

        return wordsMatrix;
    }

    /**
     * 从字库中获取指定区码和位码汉字的字模信息
     * @param areaCode 区码，对应编码的第一个字节
     * @param posCode  位码，对应编码的第二个字节
     * @return
     */
    private byte[] read(Context context, int areaCode, int posCode) {
        byte[] data = null;
        try {
            int area = areaCode - 0xa0;
            int pos = posCode - 0xa0;
            InputStream in = context.getAssets().open(DOT_MATRIX_FONT);
            int offset = ((area - 1) * 94 + pos -1) * mWordByteByDots;
            in.skip(offset);
            data = new byte[mWordByteByDots];
            in.read(data, 0, mWordByteByDots);
            in.close();
        } catch (IOException e) {
            Log.d(TAG, "IOException e = " + e.getMessage());
        }
        return data;
    }
```

2. 根据Boolean类型的二维数组绘制点阵，当Boolean值为false，表示要绘制空心圆，反之绘制实心圆。
```
    @Override
    protected void onDraw(Canvas canvas) {
        //super.onDraw(canvas);
        for (int row = 0; row < mDotMatrixFontType.getValue(); row++) {
            for (int col = 0; col < mDotMatrixFontType.getValue() * this.mWordNumber; col++) {
                if (mWordsMatrix[row][col]) {
                    canvas.drawCircle(col * (mPointSpace + mPaintRadius * 2) + mPointSpace + mPaintRadius,
                            row * (mPointSpace + mPaintRadius * 2) + mPointSpace + mPaintRadius,
                            mPaintRadius, mFillPaint);
                } else {
                    canvas.drawCircle(col * (mPointSpace + mPaintRadius * 2) + mPointSpace + mPaintRadius,
                            row * (mPointSpace + mPaintRadius * 2) + mPointSpace + mPaintRadius,
                            mPaintRadius, mHollowPaint);
                }
            }
        }
        Message message = new Message();
        message.obj = this.mScrollDirection;
        mHandler.sendMessageDelayed(message, mScrollSpeed.getValue());
    }
```

3. 通过Handler机制实现定期刷新界面，即实现滚动效果。在上面的代码最后，就是在绘制后向Handler发送一个延迟消息，从而进入到滚动循环，下面是在Handler中通过调用invalidate方法实现定期刷新，即实现滚动的效果：
```
    Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch ((Direction)msg.obj) {
                case LEFT:
                    matrixMoveToLeft();
                    invalidate();
                    break;
                case RIGHT:
                    matrixMoveToRight();
                    invalidate();
                    break;
            }
        }
    };

    private void matrixMoveToRight() {
        for (int row = 0; row < mDotMatrixFontType.getValue(); row++) {
            boolean temp = mWordsMatrix[row][mDotMatrixFontType.getValue() * mWordNumber - 1];
            System.arraycopy(mWordsMatrix[row], 0, mWordsMatrix[row], 1, mDotMatrixFontType.getValue() * mWordNumber - 1);
            mWordsMatrix[row][0] = temp;
        }
    }

    private void matrixMoveToLeft() {
        for (int row = 0; row < mDotMatrixFontType.getValue(); row++) {
            boolean temp = mWordsMatrix[row][0];
            System.arraycopy(mWordsMatrix[row], 1, mWordsMatrix[row], 0, mDotMatrixFontType.getValue() * mWordNumber - 1);
            mWordsMatrix[row][mDotMatrixFontType.getValue() * mWordNumber - 1] = temp;
        }
    }
```

# 注意事项
1. 将HZK16字库放到assets文件夹中

# 参考文档
1. [Android点阵屏效果的控件](http://qhyuang1992.com/index.php/2016/06/25/android_dian_zhen_ping_xiao_guo_de_kong_jian/)
2. [汉字库(HZK16)的使用](https://www.jdgcs.org/wiki/%E6%B1%89%E5%AD%97%E5%BA%93%28HZK16%29%E7%9A%84%E4%BD%BF%E7%94%A8)