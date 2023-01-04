Scenes:

What needs to be realized is that a certain or your own app or your own layer can always be kept at the top, even if its activity stack is not at the top

> Guess: The android system is sorted according to the layer, so it should be possible to modify the z value of the layer to achieve

## Progress one

rebuildLayerStacks: After each layer rebuild, the operation will not rebuild again, unless there is a new actionbar, status bar or new app, new activity
For example, after the launcher is started, it will rebuildLayerStacks, and subsequent operations in the launcher will not call rebuildLayerStacks

Output Layer: The list is displayed, the higher the z-index, the higher the display

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

OutputLayer->OutputLayerCompositionState-> z (The Z order index of this layer on this output, PS: This is the Z value of the outputLayer)

See how the z of the output layer is determined

## Progress two

This z is obtained through OutputLayer::editState to get OutputLayerCompositionState, through this OutputLayerCompositionState to modify, zOrder will be ++ operated in SurfaceFlinger::calculateWorkingSet, calculateWorkingSet is called after rebuildLayerStacks

```cpp
uint32_t zOrder = 0;
for (auto& layer : display->getOutputLayersOrderedByZ()) {
     auto& compositionState = layer->editState();
      ......
     // The output Z order is set here based on a simple counter.
     compositionState.z = zOrder++;
```

It is concluded that the z value here is to sort z again according to the order in getOutputLayersOrderedByZ, and getOutputLayersOrderedByZ is set in rebuildLayerStacks through `display->setOutputLayersOrderedByZ(std::move(layersSortedByZ))`;

The layer of compositionengine, the layer that saves some state, is different from the layer used for composition.

```cpp
int32_t sequence{sSequence++};
mCurrentState.z = 0;
mCurrentState.layerStack = 0;
```

These three variables determine the order between layers.

1. The first is layerstack, which can be regarded as the meaning of a group, and the layers corresponding to different layerstacks do not interfere with each other.
   There is a DisplayDevice class in SurfaceFlinger, which is used to represent the device, which can be hdmi or wifiDisplay. DisplayDevice also has an mLayerStack. When performing composition, only layers equal to the layerstack of this device can be displayed on this device.
2. The second value is z, that is, z-order, which indicates the order on the z-axis. The larger the number, the higher it is (provided that it is on a display)
3. The third is sequence, since mSequence is a static variable, the effect of increment is to set a unique and incremented sequence number for each layer

The add method of layersSortedByZ is responsible for putting the layer in the corresponding position (find the position where it should be inserted through binary search), compare the layerstack, then compare the z-order, and finally compare the sequence. When initializing, it just puts the layer into layersSortedByZ according to the sequence. In fact, The order is still not set.

```cpp
 if (layer->setLayer(s.z) && idx >= 0) {
    mCurrentState.layersSortedByZ.removeAt(idx);
    mCurrentState.layersSortedByZ.add(layer);
    flags |= eTransactionNeeded|eTraversalNeeded;
}
```

setLayer will return true as long as the set z-value is different from the previous one.
Then mCurrentState.layersSortedByZ.removeAt and mCurrentState.layersSortedByZ.add will be executed. At this time, the layer will be inserted into layersSortedByZ according to the true meaning of z. At this point, the real z-order of the layer is determined.

## Progress three

It is found that when a full-screen application is run on the top layer, the lower layer does not appear in the relative display, that is, the lower layer does not appear when the display layer list is cycled in rebuildLayerStacks, and see where it is removed

Because it is paused, it may be removed during pause, so it needs to keep the resume state

## Progress four

After keeping the resume state of the activity, modify the needsOutputLayer so that it can be in the outputlayer, this method is feasible.
The next thing to do is to make the bottom layer enough to be displayed on the top layer

Found that traverseInZOrder does not seem to be sorted according to Z? The test found that only modifying the z value does not affect the display of the relationship between the upper and lower layers (it should be that the sub-layers are sorted according to the order of the z-axis)
What is the logic of **traverseInZOrder?**

the top-level LayerVector
Display Overlays#0
Display Root#0
The other layers are sub-layers, sub-layers and sub-layers, so the solution of modifying mDrawingState.traverseInZOrder does not seem to work

## Progress five

Find a tricky way, since OutputLayersOrderedByZ and VisibleLayersSortedByZ are added in order through traverseInZOrder, then I will add another processing directly, not processing our layer for the first time, and adding it for the second time, so that our layer will be at the bottom upper layer

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

## Progress six

It is found that when running a 3D application, the layer cannot be raised to the top layer, and it is found that the Region visibleRegion is 0
See what affects this visibleRegion

The value of visibleRegion is set from outputLayerState.visibleRegion =tr.transform(layer->visibleRegion.intersect(displayState.viewport));
After testing, displayState.viewport is a normal area, and the visibleRegion of the layer is 0, and the visibleRegion of the layer is set in computeVisibleRegions

traverseInReverseZOrder, cycle in reverse, that is, cycle from the top layer. The advantage of this is that if the calculation reaches a certain Layer, the completely opaque visualization area has already occupied the entire screen, then the Layer visualization area below it can be No more calculations.
Region has the following types:

- Visible Region
- Transparent Region
- Translucent Region
- Completely opaque region (Opaque Region)
- Covered Region

## Progress seven

I found that I want to change the visibleRegion, it can be changed, but it destroys the original logic, so I still need to solve it fundamentally, and I still need to solve the problem of z sorting, and raise the position of my app

The hierarchical relationship of layers, each layer has its own childLayer, and its sub-layers will be sorted according to z. For example, the root layer is Display1, and there will be multi-level layers below. Their z values are probably like this of

- Display1(z=0)
  - Stack0(z=0)
    - Layer1(z=0)
    - Layer2(z=2)
  - Stack1(z=1)
    - Layer3(z=0)
    - Layer4(z=3)

So the z values of the final outputLayer may not be in order, because there is no correlation

Opening an app will probably have these layers

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

Therefore, according to the z value relationship above, for example, the layer I want to put on top is Layer4, and because each Stack corresponds to an app, it is necessary to obtain its parent layer according to layer4, and then increase the z value of the parent layer, so You can modify the z value before computeVisibleRegions in rebuildLayerStacks

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

This is a verification scheme. The final scheme still needs to directly modify the z value when addingChild

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
