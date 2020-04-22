## 前言
说起Android的自定义view，各位老油条肯定都不陌生了，我也是在最近重温《安卓开发艺术探索》的时候才发现的这个问题，**onMeasure的参数 MeasureSpec 到底表示了自身的属性还是父view的属性？**我问了身边的小伙伴，他们的回答都不对或者很模棱两可，所以我特意记录一下，希望小伙伴们能有个准确地认识。
## 结论先行
先说下结论，onMeasure是测量view或viewGroup时系统调用的，它的参数widthMeasureSpec和heightMeasureSpec既**不代表**父ViewGroup的宽高也**不代表**子View的宽高，它表示了**父ViewGroup对自己的期望**，至于子View实际的宽高得看setMeasuredDimension传的是什么。
```
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // ...
    }
```

## 具体分析
那么什么是父view的期望值？我们从头说起，在android绘制的过程中，view树的最顶层是ViewRootImpl，它是一个实现了ViewParent接口的特殊的类，我们可以把它看成所有View的爷爷，ViewRootImpl中又存着DecorView的实例，我们知道DecorView才是我们View树最上层的ViewGroup，系统会调用ViewRootImpl的performTraversal方法开始View树的绘制，这个方法里面会调用DecorView的measure方法，到此，View的绘制正式传递到DecorView中

好，停一下，ViewRootImpl传递给DecorView的measure方法中的参数是怎么来的？我们看一下performTraversal方法内部这一小段：
```java
    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
     // Ask host how big it wants to be
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
```
mWidth，mHeight指的是屏幕的宽高，lp是一个LayoutParams类型的参数，他表示当前window的尺寸参数，那么也就是说这里的值都是当前屏幕的参数，关键还得看getRootMeasureSpec
```java
    private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // Window can resize. Set max size for root view.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }
```
这里面会根据当前屏幕尺寸和当前window的参数进行判断

1. MATCH_PARENT： 精确模式，大小就是屏幕大小
2. WRAP_CONTENT：最大模式，最大不能超过windowSize
3. 指定数值的固定大小：精确模式，大小定死为rootDimension

那么这里的MeasureSpec确定好了就传递给DecorView的onMeasure里面了，接下来的measure过程就是一层一层遍历View树了，一般来讲，我们在ViewGroup的onMeasure中都会调用measureChild会子view进行measure，那么我们看下对应代码
```java
    protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
这里 lp 表示子view的具体宽高参数，也就是我们在XML设置的宽高值。那么根据父viewGroup的MeasureSpec和子view的宽高（先忽略padding，不影响我们分析），通过getChildMeasureSpec方法去得到子view的MeasureSpec
```java
    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //noinspection ResourceType
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```
这个方法就是核心方法，系统会根据父ViewGroup的MeasureSpec去计算子view的MeasureSpec，具体思想如下：

- 父ViewGroup是EXACTLY：
	1. 子View参数是具体值：
	子View的MeasureSpec就是childSize+EXACTLY
	2. 子View参数是MATCH_PARENT：
	子View的MeasureSpec就是parentSize+EXACTLY
	3. 子View参数是WRAP_PARENT：
	子View的MeasureSpec就是parentSize+AT_MOST
- 父ViewGroup是AT_MOST：
	1. 子View参数是具体值：
	子View的MeasureSpec就是childSize+EXACTLY
	2. 子View参数是MATCH_PARENT：
	子View的MeasureSpec就是parentSize+AT_MOST
	3. 子View参数是WRAP_PARENT：
	子View的MeasureSpec就是parentSize+AT_MOST
- 父ViewGroup是UNSPECIFIED：
	这种不管了╭(╯^╰)╮

所以才有了我们之前的结论，onMeasure参数中的MeasureSpec不是子view的参数也不是父viewGroup的参数，而是结合两者计算出来的一种理想值，他表示系统结合父ViewGroup和子View的设定情况而推算出子View应该是这样的一种假象情况，具体你遵不遵守随意

那么回想一下，如果我们的继承自View或ViewGroup的自定义View不重写onMeasure，那么它就不会支持wrap_content属性，看看默认的onMeasure干了什么：
```java
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
    
    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
```
如果自定义的子View设置wrap_content，那么根据上面总结的结论，自定义的子View的onMeasure中的MeasureSpec是parentSize + AT_MOST的组合，而上述代码并不区分AT_MOST和EXACTLY，全部把result设成parentSize，所以自然就不支持wrap_content啦

最后我给大家举个例子帮助理解，假设父ViewGroup宽高是WRAP_CONTENT，子View的宽是MATCH_PARENT，高是WRAP_CONTENT，那么子View的onMeasure中的MeasureSpec是什么呢？

答案就是：onMeasure中子View的宽为AT_MOST，高为AT_MOST，怎么样，整明白了吗