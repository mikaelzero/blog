---
layout: wiki # 使用wiki布局模板
wiki: mojito # 这是项目名
title: 顺滑的转场效果
---

<!-- more -->

## 功能列表

- 支持 Coil 图片加载器
- 支持 Glide 图片加载器
- 支持 Fresco 图片加载器
- 支持视频图片混合、GIF、图片预览
- 支持拖拽关闭
- 支持自定义页面索引指示器、进度条、Cover
- 支持原图加载策略

## 动图效果

{% swiper effect:cards %}
![](https://github.com/MikaelZero/Media/blob/master/mojito_gif_1.gif?raw=true)
![](https://github.com/MikaelZero/Media/blob/master/mojito_gif_2.gif?raw=true)
![](https://github.com/MikaelZero/Media/blob/master/mojito_gif_3.gif?raw=true)
![](https://github.com/MikaelZero/Media/blob/master/mojito_gif_4.gif?raw=true)
![](https://github.com/MikaelZero/Media/blob/master/mojito_gif_5.gif?raw=true)
![](https://github.com/MikaelZero/Media/blob/master/mojito_gif_6.gif?raw=true)
{% endswiper %}

# 开始

---

## 添加 dependencies

```gradle
allprojects {
    repositories {
        maven { url 'https://jitpack.io' }
    }
}

implementation "com.github.mikaelzero.mojito:mojito:$mojito_version"
//support long image and gif with Sketch
implementation "com.github.mikaelzero.mojito:SketchImageViewLoader:$mojito_version"

//load with coil
implementation "com.github.mikaelzero.mojito:coilimageloader:$mojito_version"
//load with glide
implementation "com.github.mikaelzero.mojito:GlideImageLoader:$mojito_version"
//load with fresco
implementation "com.github.mikaelzero.mojito:FrescoImageLoader:$mojito_version"
```

## 初始化

```kotlin
// in your application
Mojito.initialize(
    GlideImageLoader.with(this),
    SketchImageLoadFactory()
)

//or

//YourMojitoConfig:IMojitoConfig
Mojito.initialize(
    GlideImageLoader.with(this),
    SketchImageLoadFactory(),
    YourMojitoConfig()
)
```

## 开始使用

```kotlin
Mojito.with(context)
    .urls(SourceUtil.getSingleImage())
    .views(singleIv)
    .start()
```

# 使用

## RecyclerView

```kotlin
binding.recyclerView.mojito(R.id.srcImageView) {
    urls(SourceUtil.getNormalImages())
    position(position)
    mojitoListener(
        onClick = { view, x, y, pos ->
            Toast.makeText(context, "tap click", Toast.LENGTH_SHORT).show()
        }
    )
    progressLoader {
        DefaultPercentProgress()
    }
    setIndicator(NumIndicator())
}
```

## 单个 View

```kotlin
binding.longHorIv.mojito(SourceUtil.getLongHorImage())
```

## 无 View

```kotlin
 Mojito.start(context) {
    urls(SourceUtil.getNormalImages())
}
```

## 视频 View or 视频/图片 混合 View

```kotlin
Mojito.start(context) {
    urls(SourceUtil.getVideoImages(), SourceUtil.getVideoTargetImages())
    setMultiTargetEnableLoader(object : MultiTargetEnableLoader {
        override fun providerEnable(position: Int): Boolean {
            return position != 1
        }
    })
    setMultiContentLoader(object : MultiContentLoader {
        override fun providerLoader(position: Int): ImageViewLoadFactory {
            return if (position == 1) {
                ArtLoadFactory()
            } else {
                SketchImageLoadFactory()
            }
        }
    })
    position(position)
    views(recyclerView, R.id.srcImageView)
}
```

## Callback 回调

```kotlin
 abstract class SimpleMojitoViewCallback : OnMojitoListener {
    // image click
    override fun onClick(view: View, x: Float, y: Float, position: Int) {

    }

    //image long press
    override fun onLongClick(fragmentActivity: FragmentActivity?, view: View, x: Float, y: Float, position: Int) {
    }

    //end of min image to max image
    override fun onShowFinish(mojitoView: MojitoView, showImmediately: Boolean) {
    }

    //activity finish,backToMin,single click
    override fun onMojitoViewFinish() {
    }

    //when you drag your image
    override fun onDrag(view: MojitoView, moveX: Float, moveY: Float) {
    }

    //the ratio of long image when you scroll
    override fun onLongImageMove(ratio: Float) {
    }
}
```

## API

| Name                   |                                                          desc                                                           |
| :--------------------- | :---------------------------------------------------------------------------------------------------------------------: |
| url(src,target)        |                                                设置缩略图和原图 url 数据                                                |
| position               |                                                       点击的位置                                                        |
| views                  |                          1. recylclerView,imageViewId <br> 2. single view <br> 3. multi views                           |
| autoLoadTarget         | 默认为 true，如果你设置了原图的 url 并且设置了 autoLoadTarget(false)<br>你需要使用 setFragmentCoverLoader 来自定义 view |
| setProgressLoader      |                                        当你设置了 autoLoadTarget false 才会生效                                         |
| setIndicator           |                                     可以选择 NumIndicator 或者 CircleIndexIndicator                                     |
| setActivityCoverLoader |                                              自定义 Activity 的覆盖层 view                                              |
| setMultiContentLoader  |                                如果使用视频和图片混合模式，需要设置 ImageViewLoadFactory                                |

## Thanks

[sketch](https://github.com/panpf/sketch)

[BigImageViewer](https://github.com/Piasy/BigImageViewer)
