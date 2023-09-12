---
title: MPAndroidChart踩坑
urlname: mpandroidchart_qa
date: 2023/09/06
tags:
  - Android
---

### 打印 value 的值时出现 1.7E 这类文本

需要先将 value.toInt()之后再 toString

### X 轴最后一个标签不显示

axisMaximum 需要设置正确，假设要显示 5 个 X 轴，那么 axisMinimum 为 0f，axisMaximum 就需要设置为 4f 而不是 5f，否则会出现 6 个点，0 到 5 是 6 个点

### X 轴出现相同的 Label

当我请求接口之后，会更新数据，出现在于 Viewpager2 滑动后再滑动回来的情况下，出现了相同的 Label

发现在 AxisRenderer 中的 computeAxisValues 方法中，range 不对,最终发现是 setVisibleXRangeMaximum 方法导致的。

### setNoDataText 无效

注意 setNoDataText 的调用时机，我的问题出在项目框架中 Base 中没有提供 initData 之前的回调，initData 是在 resume 调用的，导致在显示出来的一帧并没有用到 setNoDataText，需要将 setNoDataText 提前到 create 等生命周期中

### setCircleColor 无效

在最新的 3.1.0 的源码中 LineChartRenderer 里，drawCircles 做了修改，增加了 DataSetImageCache，导致 `boolean changeRequired = imageCache.init(dataSet);`这段代码返回的是 false，就不会调用`imageCache.fill(dataSet, drawCircleHole, drawTransparentCircleHole);`,imageCache.fill 中会重新 drawCircle，同时也会更新 color

```kotlin
        protected boolean init(ILineDataSet set) {

            int size = set.getCircleColorCount();
            boolean changeRequired = false;

            if (circleBitmaps == null) {
                circleBitmaps = new Bitmap[size];
                changeRequired = true;
            } else if (circleBitmaps.length != size) {
                circleBitmaps = new Bitmap[size];
                changeRequired = true;
            }

            return changeRequired;
        }
```

在 init 中只判断了 circleBitmaps 是否为 null，但是如果更改了颜色的话，circleBitmaps 是不为 null 的,所以这里逻辑是有问题的,在原项目的 pull request 中也能找到解决方案 https://github.com/PhilJay/MPAndroidChart/pull/5231/files#diff-bce659457c538f8f1d5143d3694d99430338adc9ff68d7010e50562e1956fd41 ,或者直接把 changeRequired 这个判断给注释了

### MarkView 固定显示在中心

源码中并不支持总是显示 Markview，如果在 Highlight 的值为空的情况下是不绘制 markview 的，所以需要通过继承 LineChart 来重写 drawMarkers 方法

```kotlin
class AlwaysShowMarkViewLineChart : LineChart {
    constructor(context: Context?) : super(context)
    constructor(context: Context?, attrs: AttributeSet?) : super(context, attrs)
    constructor(context: Context?, attrs: AttributeSet?, defStyle: Int) : super(context, attrs, defStyle)

    override fun drawMarkers(canvas: Canvas) {
        // if there is no marker view or drawing marker is disabled
        if (mMarker == null || !isDrawMarkersEnabled) {
            if (mMarker != null) {
                mMarker.draw(canvas, 0f, 0f)
            }
            return
        }
        if (!valuesToHighlight()) {
            return
        }
        for (highlight in mIndicesToHighlight) {
            val set: IDataSet<Entry?> = mData.getDataSetByIndex(highlight.dataSetIndex)
            val e = mData.getEntryForHighlight(highlight)
            val entryIndex = set.getEntryIndex(e)

            // make sure entry not null
            if (e == null || entryIndex > set.entryCount * mAnimator.phaseX) {
                mMarker.draw(canvas, 0f, 0f)
                continue
            }
            val pos = getMarkerPosition(highlight)

            // check bounds
            if (!mViewPortHandler.isInBounds(pos[0], pos[1])) {
                mMarker.draw(canvas, 0f, 0f)
                continue
            }

            // callbacks to update the content
            mMarker.refreshContent(e, highlight)

            // draw the marker
            mMarker.draw(canvas, pos[0], pos[1])
        }
    }
}
```

并且，如果想要一直居中，还需要在自定义的 MarkerView 中做下判断

```kotlin
    override fun draw(canvas: Canvas?, posX: Float, posY: Float) {
        if (needOverrideDraw) {
            super.draw(canvas, (chartView.width - viewPortHandler.contentLeft()) / 2f + viewPortHandler.contentLeft() , chartView.height.toFloat())
        } else {
            super.draw(canvas, posX, posY)
        }
    }
```

### 重新加载数据出现闪动

场景是，原本可能有 10 条数据，本身有较为明显的曲线效果，然后重新加载了新的数据后，定位到了指定的位置，这时候要调用 centerTo 等方法来定位到某个点，这就导致会从原先的曲线变换到另一个曲线，即使使用 centerViewToAnimated 的方式，也会慢慢从一个点到另一个点，如果直接使用 centerViewTo，那么就会闪动一下，会将最新的曲线显示出来，如果和原曲线差距过大，就有明显的闪动。

解决方案是，加一个动画 animateY，也就是 phaseY 这个属性，这样在一开始绘制的时候，新曲线会直接绘制在 0 的位置并慢慢显示。
