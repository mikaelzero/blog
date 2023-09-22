---
title: Pitfalls of MPAndroidChart
urlname: mpandroidchart_qa
date: 2023/09/06
tags:
  - Android
---

### The value is displayed as '1.7E'

First convert the value to an integer using `value.toInt()` and then convert it to a string using `toString()`.

### The last label on the X-axis is not displayed


The `axisMaximum` needs to be set correctly，if we need display 5 labels on the `X-axis`, `axisMinimum` should be set to  `0f`，and `axisMaximum` should set to 4f instead of 5f，Otherwise,it will display 6 points,from 0 to 5.

### Duplicate label on the X-axis

When making a request to update the data after sliding back to the previous position in a ViewPager2, duplicate labels may appear.

This issue is related to the `computeAxisValues` method in `AxisRenderer`, where the `range` is incorrect. It is found that the issue is caused by the `setVisibleXRangeMaximum` method.

### setNoDataText is not effective

Pay attention to the timing of calling `setNoDataText`. The issue  arise if `setNoDataText` is called after the `initData` callback, which is called during the `onResume`. In this case, the `setNoDataText` may not be used in the frame that is currently displayed. To resolve this, `setNoDataText` should be called earlier in the lifecycle, such as during the `onCreate`.

### setCircleColor is not effective

In the latest 3.1.0 version of the source code for LineChartRenderer, there have been modifications in the `drawCircles()` , introducing a `DataSetImageCache`. This change causes the following code to return false: `boolean changeRequired = imageCache.init(dataSet)`. As a result, `imageCache.fill(dataSet, drawCircleHole, drawTransparentCircleHole)` is not called, and the circle color does not get updated.

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

In the init method, only the `circleBitmaps` variable is checked for null. However, if the color is changed, `circleBitmaps` will not be null. This logic is flawed. The pull request in the original project provides a solution: [GitHub Pull Request](https://github.com/PhilJay/MPAndroidChart/pull/5231/files#diff-bce659457c538f8f1d5143d3694d99430338adc9ff68d7010e50562e1956fd41). Alternatively, you can comment out the `changeRequired` check.


### MarkView fixed in the center.

The source code does not support always displaying the MarkView if the `Highlight` value is empty. Therefore, you need to override the `drawMarkers` method by extending the `LineChart` class.

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

Additionally,if you want to keep it centered,you need to check in custom `MarkerView`

```kotlin
    override fun draw(canvas: Canvas?, posX: Float, posY: Float) {
        if (needOverrideDraw) {
            super.draw(canvas, (chartView.width - viewPortHandler.contentLeft()) / 2f + viewPortHandler.contentLeft() , chartView.height.toFloat())
        } else {
            super.draw(canvas, posX, posY)
        }
    }
```

### Flash when reload data 

When reloading data and positioning to a specific point using methods like centerTo, there can be a flickering effect if the chart had a noticeable curve effect with the previous data. Even when using centerViewToAnimated, it will transition slowly from one point to another. If centerViewTo is used directly, there will be a quick flicker, revealing the new curve. If the difference between the old and new curves is significant, the flickering effect will be more pronounced.

The solution is to add an animation using `animateY`, which modifies the `phaseY` property. This way, when initially drawing the chart, the new curve will be drawn directly at the position of 0 and gradually displayed.
