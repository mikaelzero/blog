---
title: surfaceflinger修改layer层级
urlname: surfaceflinger_change_layer_level
date: 2022/11/11
tags:
  - SurfaceFlinger
  - layer
---

场景:

需要实现在系统中, 某一个或者自己的 app 或者是自己的 layer 能够一直保持在最顶层, 即使他的 activity 栈不是在最上层
猜测: 系统是根据 layer 来排序的, 所以应该可以修改 layer 的 z 值来实现

## 进展一

rebuildLayerStacks
每一个 layer rebuild 之后, 再次操作不会重新 rebuild, 除非出现新的 actionbar, 状态栏或者新的 app、新的 activity
比如 launcher 启动后, 会 rebuildLayerStacks, 之后在 launcher 中的操作都不会调用 rebuildLayerStacks

Output Layer: 显示的就是该列表, **z-index** 越高, 显示越前面

```text
 2 Layer  - Output Layer 0x72d8892d00 (Composition layer 0x736bcc0f18) (SurfaceView - com.mikaelzero.vr/com.mikaelzero.vrbrowser.VRBrowserActivity#0)
        Region visibleRegion (this=0x72d8892d28, count=1)
    [  0,   0, 5088, 2544]
      forceClientComposition=false clearClientTarget=true displayFrame=[0 0 5088 2544] sourceCrop=[0.000000 0.000000 5088.000000 2544.000000] bufferTransform=0 (0) z-index=0
      hwc: layer=0x0831 composition=DEVICE (2)
  - Output Layer 0x72d8890a00 (Composition layer 0x72e3a59c98) (NativeSFDemo#0)
        Region visibleRegion (this=0x72d8890a28, count=1)
    [  0,   0, 5088, 2544]
      forceClientComposition=false clearClientTarget=false displayFrame=[0 0 5088 2544] sourceCrop=[0.000000 0.000000 5088.000000 2544.000000] bufferTransform=0 (0) z-index=1
      hwc: layer=0x0832 composition=DEVICE (2)
```

OutputLayer->OutputLayerCompositionState-> **z** (The Z order index of this layer on this output, PS: 这是 outputLayer 的 z 值)

看看输出的 layer 的 z 是怎么判定的

## 进展二

这个 z 是通过 OutputLayer::editState 来获取到 OutputLayerCompositionState, 通过这个 OutputLayerCompositionState 来进行修改, 在 SurfaceFlinger::calculateWorkingSet 中会对 zOrder 进行++操作, calculateWorkingSet 是在 rebuildLayerStacks 之后调用

```cpp
uint32_t zOrder = 0;
for (auto& layer : display->getOutputLayersOrderedByZ()) {
     auto& compositionState = layer->editState();
      ......
     // The output Z order is set here based on a simple counter.
     compositionState.z = zOrder++;
```

得出, 这里的 z 值是根据 getOutputLayersOrderedByZ 中的顺序来再次进行 z 的排序, 而 getOutputLayersOrderedByZ 是在 rebuildLayerStacks 中通过 `display->setOutputLayersOrderedByZ(std::move(layersSortedByZ));` 来设置

compositionengine 的 layer, 保存一些 state 的 layer 和合成用的 layer 功能不一样

```cpp
int32_t sequence{sSequence++};
mCurrentState.z = 0;
mCurrentState.layerStack = 0;
```

这三个变量决定了 layer 之间的顺序.
1）首先是 layerstack, 它可以看成是组的含义, 不同的 layerstack 对应的 layer 互不干扰.
SurfaceFlinger 中有一个 DisplayDevice 类, 它用来表示设备, 可以是 hdmi 也可以是 wifiDisplay DisplayDevice 里也有个 mLayerStack, 进行 composition 的时候, 只有和这个 device 的 layerstack 相等的 layer 才可以显示到这个设备上.
2）第二个值是 z, 也就是 z-order, 表示 z 轴上的顺序, **数字越大, 表示越在上面(前提是在一个 display 上)**
3）第三个是 sequence, 由于 mSequence 是一个 static 变量, 所以递加的效果是为每个 layer 设置一个唯一且递增的序列号

layerSortedByZ 的 add 方法负责将 layer 放到对应的位置（通过二分查找找到应该插入的位置）, 比较 layerstack, 再比较 z-order, 最后比较 sequence, 初始化的时候只是根据 sequence 将 layer 放到 layersSortedByZ 而已, 其实顺序还是没有设置.

```cpp
 if (layer->setLayer(s.z) && idx >= 0) {
    mCurrentState.layersSortedByZ.removeAt(idx);
    mCurrentState.layersSortedByZ.add(layer);
    flags |= eTransactionNeeded|eTraversalNeeded;
}
```

只要设置的 z 值和之前的不同, setLayer 就会返回 true.
然后 mCurrentState.layersSortedByZ.removeAt 和 mCurrentState.layersSortedByZ.add 就会被执行, 这时, layer 将会按照 z 真正意义上插入到 layersSortedByZ 中, 到这里, layer 的真正 z-order 就确定好了

## 进展三

发现在最顶层跑一个全屏的应用时, 底下的 layer 就没有出现在相对于 display 中, 即在 rebuildLayerStacks 中循环 display 的 layer 列表时不出现底下的 layer, 看看是哪里移除了

因为 pause 了, 在 pause 的时候, 可能移除了, 所以需要保持 resume 状态

## 进展四

保持 activity 的 resume 状态后, 修改 needsOutputLayer 让它能够在 outputlayer 中, 该方法是可行的.
接下来要做的就是让底下的 layer 都够在顶层显示

发现 traverseInZOrder 好像不是按照 Z 来排序的？测试发现仅仅修改 z 值并不影响显示上下层关系(应该是子 layer 按照 z 轴顺序排序的)
**traverseInZOrder 的逻辑是什么？**

最上级的 LayerVector
Display Overlays#0
Display Root#0
其他的 layer 都是子 layer, 子 layer 还有子 layer, 所以修改 mDrawingState.traverseInZOrder 的方案看起来不太行

## 进展五

找到一个取巧的方法, 既然 OutputLayersOrderedByZ 和 VisibleLayersSortedByZ 是通过 traverseInZOrder 按顺序添加的, 那我直接再加一次处理, 第一次不处理我们的 layer, 到了第二次再添加, 这样我们的 layer 就会在最上层

```cpp
mDrawingState.traverseInZOrder([&](Layer* layer) {
    bool skip=false;
    if (layer->getName().contains("layer name")) {
       skip = true;
    }
    ......
    if (needsOutputLayer&&!skip) {
    ......

    mDrawingState.traverseInZOrder([&](Layer* layer) {
    bool needHandle = false;
    if (layer->getName().contains("layer name")) {
          needHandle = true;
    }
}
```

## 进展六

发现跑 3D 应用的时候, 无法将 layer 提到最上层, 发现是 Region visibleRegion 都为 0
看看什么会影响这个 visibleRegion

visibleRegion 的值时从 `outputLayerState.visibleRegion =tr.transform(layer->visibleRegion.intersect(displayState.viewport));` 设置的
经过测试, displayState.viewport 是正常的区域, 而 layer 的 visibleRegion 都为 0, layer 的 visibleRegion 是在 computeVisibleRegions 中进行设置的

traverseInReverseZOrder, 反向地循环, 也就是从最顶层循环, 这样做的好处是, 如果计算到某一层 Layer 时, 完全不透明的可视化区域已经占满整个屏幕, 那么这之下的 Layer 可视化区域就可以不用计算了.
Region 有下面几种类型：

1. 可见区域（Visible Region）
2. 透明区域（Transparent Region）
3. 半透明区域（Translucent Region）
4. 完全不透明区域（Opaque Region）
5. 被覆盖区域（Covered Region）

来自:[computeVisibleRegions](https://blog.csdn.net/u014535072/article/details/106794482)

```cpp
void SurfaceFlinger::computeVisibleRegions(const sp<const DisplayDevice>& displayDevice,
                                           Region& outDirtyRegion, Region& outOpaqueRegion) {
    ATRACE_CALL();
    ALOGV("computeVisibleRegions");

    auto display = displayDevice->getCompositionDisplay();

    Region aboveOpaqueLayers;
    Region aboveCoveredLayers;
    Region dirty;

    outDirtyRegion.clear();

    // 先找到“感兴趣的”Layer, 也就是这个layer是属于SecureDisplay的
    // 暂时没有找到相关的说明, 忽略好了
    Layer* layerOfInterest = NULL;
    bool bIgnoreLayer = false;
    mDrawingState.traverseInReverseZOrder([&](Layer* layer) {
        if (layer->isSecureDisplay()) {
            bIgnoreLayer = true;
            if (displayDevice->isPrimary()) {
                layerOfInterest = layer;
            }
            return;
        }
    });

    // 反向遍历Z轴计算可视化区域
    mDrawingState.traverseInReverseZOrder([&](Layer* layer) {
        // 获取当前绘制中的Surface
        const Layer::State& s(layer->getDrawingState());

        // 只考虑给定图层堆栈上的layer
        if (!display->belongsInOutput(layer->getLayerStack(), layer->getPrimaryDisplayOnly())) {
            return;
        }

        // 忽略SecureDisplay中的layer
        if (bIgnoreLayer && layerOfInterest != layer) {
            Region visibleNonTransRegion;
            visibleNonTransRegion.set(Rect(0, 0));
            layer->setVisibleNonTransparentRegion(visibleNonTransRegion);
            return;
        }

        // 完全不透明的Surface区域
        Region opaqueRegion;

        // 在屏幕上可见且不完全透明的Surface区域.
        // 这实际上是该层的足迹减去其上方的不透明区域.
        // 半透明Surface覆盖的区域被认为是可见的.
        Region visibleRegion;

        // 被其上方所有可见区域覆盖的Surface区域（包括半透明区域）.
        Region coveredRegion;

        // 暗示完全透明的表面区域.  这仅用于告诉图层何时没有可见的非透明区域, 可以将其从图层列表中删除.
        // 它不会影响此层或它下面的任何层的visibleRegion.
        // 如果应用程序不遵守SurfaceView限制（不幸的是, 有些不遵守）, 则提示可能不正确.
        Region transparentRegion;

        // 处理不可见或者被隐藏的Surface的方式就是将其可视化的区域设置为空
        if (CC_LIKELY(layer->isVisible())) {
            // 如果该Surface不是完全不透明的, 则视为半透明
            const bool translucent = !layer->isOpaque(s);
            Rect bounds(layer->getScreenBounds());

            // 当前Surface的可视区域默认为屏幕大小或者Surface在屏幕中的大小
            visibleRegion.set(bounds);
            ui::Transform tr = layer->getTransform();

            // Region为空则说明没有可视区域
            // 注意 Region 是一个矩形（Rect）集合
            if (!visibleRegion.isEmpty()) {
                // 首先从可见区域移除透明区域
                if (translucent) {
                    // 函数preserveRects的返回值为false
                    // 说明需要忽略掉当前正在处理的应用程序窗口的透明区域
                    if (tr.preserveRects()) {
                        // 标记透明区域, 这个透明区域就是transparentRegionHint遍历
                        // 在 SurfaceFlinger.setClientStateLocked过程中设置的
                        transparentRegion = tr.transform(layer->getActiveTransparentRegion(s));
                    } else {
                        // 转换太复杂, 无法进行透明区域优化.
                        transparentRegion.clear();
                    }
                }

                // 计算不透明区域
                const int32_t layerOrientation = tr.getOrientation();
                if (layer->getAlpha() == 1.0f && !translucent &&
                        layer->getRoundedCornerState().radius == 0.0f &&
                        ((layerOrientation & ui::Transform::ROT_INVALID) == false)) {

                    // 当当前正在处理的应用程序窗口是完全不透明, 并且旋转方向也是规则时
                    // 那么它的完全不透明区域opaqueRegion就等于计算所得到的可见区域visibleRegion
                    opaqueRegion = visibleRegion;
                }
            }
        }

        // 该Surface没有可视区域, 则清空相关变量, 直接返回
        if (visibleRegion.isEmpty()) {
            layer->clearVisibilityRegions();
            return;
        }

        // 将覆盖区域裁剪到可见区域
        // aboveCoveredLayers用来描述当前正在处理的应用程序窗口的所有上层应用程序窗口所组成的可见区域
        // 将这个区域与当前正在处理的应用程序窗口的可见区域visibleRegion相交, 就可以得到当前正在处理的应用程序窗口的被覆盖区域coveredRegion
        // 而将这个区域与当前正在处理的应用程序窗口的可见区域visibleRegion相或一下, 就可以得到下一个应用程序窗口的所有上层应用程序窗口所组成的可见区域aboveCoveredLayers.
        coveredRegion = aboveCoveredLayers.intersect(visibleRegion);

        // aboveOpaqueLayers用来描述当前正在处理的应用程序窗口的所有上层应用程序窗口所组成的完全不透明区域
        aboveCoveredLayers.orSelf(visibleRegion);

        // 这个区域从当前正在处理的应用程序窗口的可见区域visibleRegion减去后, 就可以得到当前正在处理的应用程序窗口的最终可见区域visibleRegion.
        visibleRegion.subtractSelf(aboveOpaqueLayers);

        // 计算Layer的脏区域, 所谓脏区域就是需要重新执行渲染操作的
        if (layer->contentDirty) {
            // 成员变量contentDirty的值为true, 则说明当前正在处理的Layer上一次的状态还未来得及处理
            // 即它当前的内容是脏的. 在这个状况下, 只需要将此次的可见区域与上一次的可见区域合并即可
            dirty = visibleRegion;
            // as well, as the old visible region
            dirty.orSelf(layer->visibleRegion);
            layer->contentDirty = false;
        } else {

            // 当上一次状态已经处理了, 也就是显示内容没有更新,则无需重新渲染所有区域.
            // 现在只需要处理一下两种情况：
            // 1. 之前是被覆盖的区域, 但现在不被覆盖了
            // 2. 由于窗口大小变化而引发的新增不被覆盖区域

            // 针对第一种情况:
            // 将当前可见区域visibleRegion与它的上一次被覆盖区域oldCoveredRegion相交
            // 就可以得到之前是被覆盖的而现在不被覆盖了的区域, 即可以得到第一部分需要重新渲染的区域
            // 上一次可见区域和被覆盖区域分别oldVisibleRegion, oldCoveredRegion

            // 针对第二种情况:
            // 由于将一个应用程序窗口的当前可见区域减去被覆盖区域即为它的当前不被覆盖的区域newExposed
            // 同理上一次不被覆盖的区域oldExposed就是上一次可见区域减去上一次被覆盖区域
            // 那么将一个应用程序窗口的当前不被覆盖的区域newExposed减去它的上一次不被覆盖的区域oldExposed, 就可以得到新增的不被覆盖区域
            const Region newExposed = visibleRegion - coveredRegion;
            const Region oldVisibleRegion = layer->visibleRegion;
            const Region oldCoveredRegion = layer->coveredRegion;
            const Region oldExposed = oldVisibleRegion - oldCoveredRegion;

            // 将第一部分和第二部分需要重新渲染的区域组合起来, 就可以得到当前Layer的脏区域dirty.
            dirty = (visibleRegion&oldCoveredRegion) | (newExposed-oldExposed);
        }

        // 从该脏区域dirty减去上层的完全不透明区域
        // 因为后者的渲染不需要当前Layer参与
        dirty.subtractSelf(aboveOpaqueLayers);

        // 新的脏区域dirty累计到输出参数dirtyRegion中.
        outDirtyRegion.orSelf(dirty);

        // 更新计算到目前为止所得到的Layer的完全不透明区域
        // 这个是方便下一层Layer的计算
        aboveOpaqueLayers.orSelf(opaqueRegion);

        // 保存当前正在处理的Layer的可见区域和被覆盖区域以及可见非透明区域.
        layer->setVisibleRegion(visibleRegion);
        layer->setCoveredRegion(coveredRegion);
        layer->setVisibleNonTransparentRegion(
                visibleRegion.subtract(transparentRegion));
    });

    // 将前面所有的Layer组成的完全不透明区域aboveOpaqueLayers保存在输出参数opaqueRegion中
    outOpaqueRegion = aboveOpaqueLayers;
}
```

看了下 pico 的层级结构, 应该是把所有的主屏的应用都创建为自己的 layer, 这样可以自己管理所有的子 layer, 所以这是一个最终的方案

## 进展七

发现想改 visibleRegion, 改是能改, 但是破坏了原本的逻辑, 所以还是要从根本上解决, 还是要解决 z 排序的问题, 把自己的 app 的位置提上去

layer 的层级关系, 每个 layer 都有自己的 childLayer, 自己的子 layer 才会按照 z 来排序, 比如最根部的一个 layer 为 Display1, 下面会有多级的 layer, 那他们的 z 值大概是这样的

- Display1(z=0)
  - Stack0(z=0)
    - Layer1(z=0)
    - Layer2(z=2)
  - Stack1(z=1)
    - Layer3(z=0)
    - Layer4(z=3)

所以最终的 outputLayer 他们的 z 值有可能不是按照顺序的, 因为没有关联

打开一个 app 大概会有这些 layer

```cpp
layer name = Stack=1#0 z = 3 parent name: com.android.server.wm.DisplayContent$TaskStackContainers@ddb12c9#0
layer name = animation background stackId=1#0 z = 0 parent name: Stack=1#0
layer name = Task=190#0 z = 0 parent name: Stack=1#0
layer name = AppWindowToken{4c61e2d token=Token{d9dd44 ActivityRecord{ead5e57 u0 com.mikaelzero.vr/com.mikaelzero.vrbrowser.VRBrowserActivity t190}}}#0 z = 0 parent name: Task=190#0
layer name = 3bfd3d2 com.mikaelzero.vr/com.mikaelzero.vrbrowser.VRBrowserActivity#0 z = 0 parent name: AppWindowToken{4c61e2d token=Token{d9dd44 ActivityRecord{ead5e57 u0 com.mikaelzero.vr/com.mikaelzero.vrbrowser.VRBrowserActivity t190}}}#0
layer name = Background for -SurfaceView - com.mikaelzero.vr/com.mikaelzero.vrbrowser.VRBrowserActivity#0 z = -2147483648 parent name: SurfaceView - com.mikaelzero.vr/com.mikaelzero.vrbrowser.VRBrowserActivity#0
layer name = Bounds for - com.mikaelzero.vr/com.mikaelzero.vrbrowser.VRBrowserActivity#0 z = -2 parent name: com.mikaelzero.vr/com.mikaelzero.vrbrowser.VRBrowserActivity#0
layer name = SurfaceView - com.mikaelzero.vr/com.mikaelzero.vrbrowser.VRBrowserActivity#0 z = -2 parent name: com.mikaelzero.vr/com.mikaelzero.vrbrowser.VRBrowserActivity#0
layer name = com.mikaelzero.vr/com.mikaelzero.vrbrowser.VRBrowserActivity#0 z = 0 parent name: 3bfd3d2 com.mikaelzero.vr/com.mikaelzero.vrbrowser.VRBrowserActivity#0
```

因此, 根据上面的 z 值关系, 比如我想要置顶的 layer 是 Layer4, 又因为每一个 Stack 对应一个 app, 所以需要根据 layer4 获取到他的父 layer, 然后把这个父 layer 的 z 值提高, 所以可以在 rebuildLayerStacks 中, 在 computeVisibleRegions 之前修改 z 值

```cpp
mDrawingState.traverseInZOrder([&](Layer* layer) {
    if (layer->getName().contains("needTopLayerName") &&
        layer->getName().contains("ActivityRecord")
        ) {
        sp<Layer> stackLayer = layer->getParent()->getParent();
        sp<Layer> displayLayer = stackLayer->getParent();
        if (displayLayer) {
            displayLayer->setChildLayer(stackLayer, INT_MAX);
        }
    }
});
```

这是一个验证的方案, 最终方案还是需要在 addChild 的时候, 直接修改 z 值即可

```cpp
void Layer::addChild(const sp<Layer>& layer) {
    mChildrenChanged = true;
    setTransactionFlags(eTransactionNeeded);

    mCurrentChildren.add(layer);
    layer->setParent(this);
    if (layer->getName().contains("com.mikaelzero.layername") &&
        layer->getName().contains("ActivityRecord")) {
        if (layer->getParent()) {
            sp<Layer> stackLayer = layer->getParent()->getParent();
            if (stackLayer && stackLayer->getName().contains("Stack=")) {
                if (stackLayer->getZ() != INT_MAX) {
                    stackLayer->setLayer(INT_MAX);
                }
            }
        }
    }
}
```
