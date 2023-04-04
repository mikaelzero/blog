---
title: 纸箱子Cardboard上也能手势操作？
urlname: gestures-be-operated-on-the-cardboard
date: 2023/03/30
tags:
  - VR
  - Cardboard
---

**Cardboard**是 Google 推出的一款超轻量级的 VR 眼镜，只需要将你的手机放入这个小纸盒子中，就可以看到 VR 效果(虽然效果一般，毕竟两个透镜是比较差的)。我们可以从官方的几张图来看下**Cardboard**

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/hiVMOb.png)

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/2UqTGe.jpg)

也有一些很骚气的配色

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/Si9tY7.png)

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/81kyOi.png)

并且它是可定制的，它提供了图纸，你可以自己选择材料来做定制。

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/HmNSKr.png)

**Cardboard**本身不附带任何功能，仅仅是一个眼镜纸壳，观看 VR 都是依靠手机的性能和传感器来实现。

关于手势操作 Google 也提供了两个方案：
![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/46pYeH.png)

左边的就是侧边有一个磁铁，这个磁铁可以上下拨动，通过手机的磁场传感器来识别磁场的变换，来实现用户的上下识别或者是是非值识别，右边则是通过点按金属薄片来识别用户手势。但是这两个基本都是能识别「是」或者是「否」的操作，没办法识别更高级的操作。

现有的一些方案，也是比较有趣的：

1. 在纸盒子中加入眼动追踪，根据眼球来识别操作。
2. 通过对纸盒高度定制化，比如把纸盒子抠出一些区域来装上一些滑动或者是圆形装置。
3. 连上耳机，根据耳机在 3D 场景下的一个变换来判断用户的操作。

等等等等方案，大多数都是要基于其他的硬件设备或者是定制化才能够实现，那么当我们身边只有一个**Cardboard**，如何实现手势的功能呢？

参考上述的第二个方案，其实我们可以直接根据手机的麦克风来实现，手机的麦克风一般有两个，顶部的降噪麦克风和底部的通话麦克风

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/B8tUOq.png)

如果动作相同方向不同，在前面或者左右边做相同的动作，通过左右两个声道的差异来做，两个麦克风在手机上一般不会是一条直线，会形成一个高度差，可以通过这个高度差来分辨是左还是右。

这个方案首先需要确定手势的路径，比如用户作出什么样的动作是作为什么样的手势和功能，需要做出一堆的数据集。

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/sNFB60.png)

其次就是算法，根据麦克风和左右声道做一些特征然后分类出手势(我不是很懂这个算法，只能讲到这了)。

另外由于训练集很大，当前的手机性能还撑不起这样的计算，所以只能放到云端，而云端必然会有延迟，这个方案的延迟大概有 2s 左右的延迟，当然对于一个只有纸盒子的场景来说，也不是不能接受。

后续猜想：这是一个很有趣的解决方案，你可以通过手势完成和 VR 高端眼镜相同的各种操作，甚至还可以玩游戏，虽然延迟很高。
而且后续也可以接入手柄再配合手势来玩游戏，也一定有趣。

参考：文章内容思路和设计均来源于[GAMES Webinar-199-Taizhou Chen-VR 专题](https://www.bilibili.com/video/BV1FQ4y1r7hk/?spm_id_from=333.1007.top_right_bar_window_default_collection.content.click&vd_source=f2663ea59eb178148105cde52cff7c29)
