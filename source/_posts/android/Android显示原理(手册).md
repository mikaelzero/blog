---
title: Android显示原理(手册)
urlname: android_display_principle_manual
date: 2022/06/06
tags:
  - android
  - graphic
---

## AMS

### ActivityRecord

在 AmS 中使用 ActivityRecord 类来保存一个 Activity 相关的信息, ActivityRecord 类本身是一个 Binder 类, 名称为 IApplicationToken.Stub. 每个 ActivityRecord 都会在 WmS 中对应一个 AppWindowToken 类, 该类保存了和 ActivityRecord 相关的所有窗口信息, 比如启动窗口、实际窗口、关闭窗口等. 当启动一个新的 Activity 时, AmS 中会先创建一个 ActivityRecord 对象, 并请求 WmS 中也创建一个 AppWindowToken 对象;当销毁一个 Activity 时, AmS 会请求 WmS 删除 AppWindowToken 对象.

## WMS

### 第一类:窗口管理相关

```java
final HashSet<Session> mSessions;
final HashMap<IBinder, WindowState> mWindowMap; final ArrayList<WindowState> mWindows; //Z-ordered final HashMap<IBinder, WindowToken> mTokenMap =
new HashMap<IBinder, WindowToken>();
final ArrayList<WindowToken> mTokenList;
final ArrayList<AppWindowToken> mAppTokens; //Z-ordered
//\* Windows whose animations have ended and now must be removed, final ArrayList<WindowState> mPendingRemove
// Windows whose surface should be destroyed,
final ArrayList<WindowState> mDestroySurface;
ArrayList<WindowState> mForceRemoves;
```

### 第二类, 窗口动画相关

```java
// Window tokens that are in the process of exiting, but still
II on screen for animations.
final ArrayList<WindowToken> mExitingTokens
final ArrayList<AppWindowToken> mExitingAppTokens; final ArrayList<AppWindowToken> mFinishedStarting ;
 /**
  * This was the app token that was used to retrieve the last enter _ animation. It will be used for the next exit animation.
**/
  AppWindowToken mLastEnterAnimToken;
//These were the layout params used to retrieve the last enter animation. _ They will be used for the next exit animation.
  LayoutParams mLastEnterAnimParams;
  // Windows whose animations have ended and now must be removed.
  final ArrayList<WindowState> mPendingRemove;
  // State management of app transitions.
  int mNextAppTransition = WindowManagerPolicy.TRANSIT_UNSET;
  String mNextAppTransitionPackage;
  int mNextAppTransitionEnter;
  int mNextAppTransitionExit;
  boolean mAppTransitionReady = false;
  boolean mAppTransitionRunning = false;
  boolean mAppTransitionTimeout = false;
   boolean mStartinglconlnTransition = false;
  boolean mSkipAppTransitionAnimation = false;
  final ArrayList<AppWindowToken> mOpeningApps;
  final ArrayList<AppWindowToken> mClosingApps;
  final ArrayList<AppWindowToken> mToTopApps;
  final ArrayList<AppWindowToken> mToBottomApps;
  float mWindowAnimationScale = l.Of;
  float mTransitionAnimationScale = 1.0f;
```

### 第三类, 输出法窗口管理相关

```java
 InputMethodManager mlnputMethodManager;
  final ArrayList<WindowState> mResizingWindows;
  // This just indicates the window the input method is on top of, not // necessarily the window its input is going to.
  WindowState mlnputMethodTarget = null;
  WindowState mUpcominglnputMethodTarget = null;
  boolean mlnputMethodTargetWaitingAnim;
  int mlnputMethodAnimLayerAdjustment;
  WindowState mlnputMethodWindow = null;
  final ArrayList<WindowState> mlnputMethodDialogs;
```

### 第四类, 墙纸窗口管理相关

```java
  final ArrayList<WindowToken> mWallpaperTokens;
  // If non-null, this is the currently visible window that is associated
  // with the wallpaper.
  WindowState mWallpaperTarget = null;
  // If non-null, weare in the middle of animating from one wallpaper target // to another, andthis is the lower one in Z-order.
  WindowState mLowerWallpaperTarget = null;
  // If non-null, weare in the middle of animating from one wallpaper target // to another, andthis is the higher one in Z-order.
  WindowState mUpperWallpaperTarget = null;
  int mWallpaperAnimLayerAdjustment;
  float mLastWallpaperX = -1;
  float mLastWallpaperY = -1;
  float mLastWallpaperXStep = -1;
  float mLastWallpaperYStep = -1;
  // This is set when we are waiting for a wallpaper to tell us it is done
  // changing its scroll position.
  WindowState mWaitingOnWallpaper;
  // The last time we had a timeout when waiting for a wallpaper.
  long mLastWallpaperTimeoutTime;
  //We give a wallpaper up to 150ms to finish scrolling.
  static final long WALLPAPER_TIMEOUT = 150;
  // Time we wait after a timeout before trying to wait again.
  static final long WALLPAPER_TIMEOUT_RECOVERY = 10000;
```

### 第五类, 焦点窗口管理相关

```java
  //_ Windows that have lost input focus and are waiting for the new
  //_ focus window to be displayed before they are told about this.
  ArrayList<WindowState> mLosingFocus = new ArrayList<WindowState>();
   WindowState mCurrentFocus = null;
  WindowState mLastFocus = null;
  AppWindowToken mFocusedApp = null;
```

### WindowContainer

WindowContainer 用于作为一个 Window 容器, 用于管理添加进来的子 WindowContainer. 在 WMS 中存在以下几种 WIndowContainer：

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/uFPpZp.jpg)

WindowContainer 作为这些 Container 的父类, 主要提供对 child container 的添加操作：

```java
    protected void addChild(E child, Comparator<E> comparator) {
        if (child.getParent() != null) {
            throw new IllegalArgumentException("addChild: container=" + child.getName()
                    + " is already a child of container=" + child.getParent().getName()
                    + " can't add to container=" + getName());
        }

        int positionToAdd = -1;
        if (comparator != null) {
            final int count = mChildren.size();
            for (int i = 0; i < count; i++) {
                if (comparator.compare(child, mChildren.get(i)) < 0) {
                    positionToAdd = i;
                    break;
                }
            }
        }

        if (positionToAdd == -1) {
            mChildren.add(child);
        } else {
            mChildren.add(positionToAdd, child);
        }
        onChildAdded(child);

        // Set the parent after we've actually added a child in case a subclass depends on this.
        child.setParent(this);
    }
```

addChild 根据该 container 的 comparator 比较 mChildren 中的 child, 找到指定插入位置, 可以使得 mChildren 中的 E 按照一定顺序排列, 并且 child 能够直接找到其父 container.

1、DisplayContent
其对应于一个显示屏的容器, 但是其不可能存在子屏, 所以其不能直接 addChild. 具体参见 android wms——DisplayContent 中内容.

2、WindowState
用于控制每一个 window 的状态, 包含计算当前窗体的大小, 以及 session 的绑定. 其添加 child 是按照 mSubLayer 子序排序.

3、WindowToken
作为 window 句柄, 其子成员为 WindowState, 只核心变量成员 IBinder token 对应于 APP 端的 Window. 其子成员 windowState 的添加策略是根据该 windowState.mBaseLayer 主序（也就是 window 的类型）从小到大排序. 具体参见 android WMS—— WindowToken.

4、Task
Activity 对应的栈, 其子 container 为 AppWindowToken, 因此实际上为控制 APP 端 Activity 对应的 AppWindowToken 的 List 集合. 其成员变量中包含 mTaskId.

5、TaskStack
子 Container 为 task, 其实际为 WMS 中管理 task 的数据结构.

```java
    private WindowContainer<WindowContainer> mParent = null;
    protected final WindowList<E> mChildren = new WindowList<E>();
```

1. 首先是一个同样为 WindowContainer 类型的 mParent 成员变量, 保存的是当前 WindowContainer 的父容器的引用.
2. 其次是 WindowList 类型的 mChildren 成员变量, 保存的则是当前 WindowContainer 持有的所有子容器. 并且列表的顺序也就是子容器出现在屏幕上的顺序, 最顶层的子容器位于队尾.

### Window 类

该类在 android.view 包中, 是一个 abstract 类, 该类是对包含有可视界面的窗口的一种包装. 所谓的可视界面就是指各种 View 或者 ViewGroup.

### ViewRootImpl 类

该类在 android.view 包中, 客户端申请创建窗口时需要一个客户端代理, 用以和 WmS 进行交互, 这个就是 ViewRootImpl 的功能, 每个客户端的窗口都会对应一个 ViewRootImpl 类.

### W 类

该类是 ViewRootImpl 类的一个内部类, 继承于 Binder,用于向 WmS 提供一个 IPC 接口, 从而让 WmS 控制窗口客户端的行为.

## token

### token 的含义

token,该变量的类型一般都是一个 IBinder 对象,即为了进行 IPC 调用.
而与创建窗口相关的 IPC 对象一般只有两种, 1 种是 指向某个 W 类的 token,另一种是指向某个 ActivityRecord 的 token. 其中 ActivityRecord 对象是 AmS 内 部为运行的每一个 Activity 创建的一个 Binder 对象, 客户端的 Activity 可以通过该 Binder 对象通知当前 Activity 的状态.

### Activity 中的 mToken

AmS 内部为每一个运行的 Activity 都创建了一个 ActivityRecord 对象, 该对象的基类是 一个 Binder,因此 mToken 变量的意义正是指向了该 ActivityRecord 类. 该变量的值是在 Activity.init() 函数中完成的

### Window 中的 mAppToken

每一个 Window 对象中都有一个 mAppToken 变量, 注意这里说的是 Window 对象, 而不是窗口. 前面说过, 一个窗口本质上是一个 View, 而 Window 类却是一个应用窗口的抽象, 这就好比 Window 侧重于一个窗口的交互, 而窗口(View)则侧重于窗口的显示. 所以, mAppToken 并不是 W 类的引用, 事实上正如其名称所指, 它是 AmS 在远程为每一个 Activity 创建的 HistoryRecord 的引用.
事实上, Window 类中还有其他的 Binder 对象, 同时由于 Window 并不一定要对应一个 Activity,因此,如果 Window 类不属于某个 Activity , mAppToken 的变量则为空 ,否则 mAppToken 的值与 Activity 中的 mToken 值是相同的.

### WindowManager.LayoutParams 中的 token

WindowManager.LayoutParams 中 token 的意义正如其所在的类的名称, 该类是在添加窗口时指定该 窗口的布局参数, 而 token 的意义正是指定该窗口对应的 Binder 对象, 因为 WmS 需要该 Binder 对象, 以便对客户端进行 IPC 调用.
那么该 Binder 对象应该是谁呢?你可能会认为该 token 应该是 W 类, 没错, 这只是一种情况而 已, 具体来讲, 该 token 变量的值可以有三种.

- 如果创建的窗口是应用窗口, token 的值和 Window 中 mAppToken 值相同
- 如果创建的窗口为子窗口, token 为其父窗口的 W 对象.
- 如果创建的窗口是系统窗口, 那么, token 值为空.

### View 中的 token

其含义是当该 View 对象被真正作为某个窗口 W 类的内部 View 时, 该 变量就会被赋值为 ViewRoot 中的 mAttachlnfo.

## 窗口和 Layer(图层)

### ANativeWindow

是对一个本地窗口的抽象描述 其定义位于:[/frameworks/native/libs/nativewindow/include/system/window.h](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/nativewindow/include/system/window.h) 定义了比如 query、queueBuffer 和 dequeueBuffer 等等方法

### ANativeWindowBuffer

主要是定义了有关 buffer 的宽、高、格式等信息, GraphicBuffer 继承了 ANativeWindowBuffer

### Surface

Surface 继承了 ANativeWindow, 并对其中的功能做了具体实现 位于 [/frameworks/native/libs/gui/include/gui/Surface.h](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/include/gui/Surface.h)

```cpp
Surface::Surface(const sp<IGraphicBufferProducer>& bufferProducer, bool controlledByApp,
                 const sp<IBinder>& surfaceControlHandle)
  : .... {
    // Initialize the ANativeWindow function pointers.
    ANativeWindow::setSwapInterval  = hook_setSwapInterval;
    ANativeWindow::dequeueBuffer    = hook_dequeueBuffer;
    ANativeWindow::cancelBuffer     = hook_cancelBuffer;
    ANativeWindow::queueBuffer      = hook_queueBuffer;
    ANativeWindow::query            = hook_query;
    ANativeWindow::perform          = hook_perform;

    ANativeWindow::dequeueBuffer_DEPRECATED = hook_dequeueBuffer_DEPRECATED;
    ANativeWindow::cancelBuffer_DEPRECATED  = hook_cancelBuffer_DEPRECATED;
    ANativeWindow::lockBuffer_DEPRECATED    = hook_lockBuffer_DEPRECATED;
    ANativeWindow::queueBuffer_DEPRECATED   = hook_queueBuffer_DEPRECATED;
}
```

Surface 是一块绘制区域的抽象, 它对应着 Server 服务端 Surfacelinger 中的一个图层 Layer, 这个图层的背后是一块图形缓冲区 GraphicBuffer, Client 客户端的应用程序的 UI 使用软件绘制、硬件绘制在 Surface 上各种渲染操作时, 绘制操作的结果其实也就是在该图形缓冲区中.createSurface=createLayer(通常是 BufferQueueLayer)

### SurfaceControl:

用于控制 surface 的一个类, 与 surfaceflinger 通信, 有成员类 SurfaceComposerClient

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/3GPAZA.jpg)

### SurfaceComposerClient

每一个应用程序都要和 SurfaceFlinger 通信, 为了能够区分哪一个应用程序, 所以有了 SurfaceComposerClient, 为 binder 的客户端

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/GvHWcS.jpg)

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/vuFilo.png)

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/vMpmxO.jpg)

### Surface, Window, View、Layer

Window -> DecorView-> ViewRootImpl -> WindowState -> Surface -> Layer 是一一对应的.
一般的 Activity 包括的多个 View 会组成 View hierachy 的树形结构, 只有最顶层的 DecorView, 也就是根结点视图, 才是对 WMS 可见的, 即有对应的 Window 和 WindowState.
应用程序进程和 WMS 服务中的两个 Surface 对象区别
就是位于应用程序进程这一侧的 Surface 对象负责绘制应用程序窗口的 UI, 即往应用程序窗口的图形缓冲区填充 UI 数据, 而位于 WMS 服务这一侧的 Surface 对象负责设置应用程序窗口的属性, 例如位置、大小等属性.
绘制应用程序窗口是独立的, 由应用程序进程来完即可, 而设置应用程序窗口的属性却需要全局考虑, 即需要由 WMS 服务来统筹安排

## DMS

### DisplayContent

DisplayContent 用于管理屏幕, 一块屏幕对应一个 DisplayContent 对象, 虽然手机只有一个显示屏, 但是可以创建多个 DisplayContent 对象, 如投屏时, 可以创建一个虚拟的 DisplayContent.

### DisplayState

这个结构体是在 Client 端(即 App 侧)定义的, 里面描述了 Client 端关于 Display 所有状态的集合, 包括了 Display 的方向, Display 里 Surface 改变, LayerStack 改变等(对应了上面的 enum 变量), what 是状态的集合, 所有的状态可以通过 "与" 操作合并到一起(仔细看上面上面的 enum 变量的值, 每一个状态都占用了十六进制的一位)

### DisplayDeviceState

DisplayDeviceState 是在 Server 端(即 SurfaceFlinger 侧)定义的, 和 DisplayDevice 很像, 内部成员也十分地类似
这两个类其实是 App 侧和 SurfaceFlinger 侧对于 Display 状态 的不同表示, SurfaceComposerClient:: Transaction::apply() 的作用一个就是将 DisplayState 传递给 DisplayDeviceState

### DisplayToken

DisplayState 和 DisplayDeiveState 都是需要跟具体 Display 设备(不管是否是 VirtualDisplay)绑定. 而 DisplayToken 就是这些 state 类型跟具体 Display 设置连接的桥梁. DisplayToken 其实只是一个 IBinder 类型的变量, 并且其值本身是没有意义的, 只是用来做索引罢了

### State

State 是 SurfaceFlinger 这个类里面的一个内部类:

```cpp
    class State {
    public:
        explicit State(LayerVector::StateSet set) : stateSet(set), layersSortedByZ(set) {}
        State& operator=(const State& other) {
            // We explicitly don't copy stateSet so that, e.g., mDrawingState
            // always uses the Drawing StateSet.
            layersSortedByZ = other.layersSortedByZ;
            displays = other.displays;
            colorMatrixChanged = other.colorMatrixChanged;
            if (colorMatrixChanged) {
                colorMatrix = other.colorMatrix;
            }
            return *this;
        }

        const LayerVector::StateSet stateSet = LayerVector::StateSet::Invalid;
        LayerVector layersSortedByZ;
        DefaultKeyedVector< wp<IBinder>, DisplayDeviceState> displays;

        bool colorMatrixChanged = true;
        mat4 colorMatrix;

        void traverseInZOrder(const LayerVector::Visitor& visitor) const;
        void traverseInReverseZOrder(const LayerVector::Visitor& visitor) const;
    };
```

displays 成员, 他是一个 <DefaultKeyedVector> 类型(Android 自定义的一个类型, 与 std::map 类似), key-value 就是我们前面都有提到的 DisplayToken 和 DisplayDeviceState

## 服务之间的关系

### WMS 与 IMS 的关系

WmS 的源码虽说超过一万行, 但其功能可归纳为两个. 第一个是保持窗口的层次关系, 以便 SurfaceFlinger 能够据此绘制屏幕;第二个是把窗口信息传递给 InputManager 对象, 以便 InputDispatcher 能够把输入消息派发给和屏幕上显示一致的窗口.

关于第一点很容易理解, 而关于第二点, 有必要特别说明一下. 首先做这样一个场景假设, 试想, 当用户闭着眼睛在手机屏幕上操作时, 谁能确定屏幕上显示的内容和用户想象的一致?当然, 因为用户随时可能会睁开眼睛, 因此 WmS 不能够作假, 只能如实显示正确的内容. 大家是否能够想象, 当闭上眼睛时, 用户以为自己在和一个音乐播放器进行交互, 并且也飘来了悦耳的音乐, 然而此时屏幕上完全可能显示的不是音乐播放器的界面, 而是一只愤怒的小鸟的照片. 如果让我们实现这样的场景, 该如何去做?

以上场景正是 InputManager 和 WmS 之间关系的体现, 如要实现以上恶作剧场景, 只需要让 WmS 给 InputManager 传递正确地窗口信息, 而在 WmS 中却保存错误的窗口信息. 这样一来, 尽管用户闭着眼睛, 但是输入的信息却能够被 InputDispatcher 正确地派发给对应的应用程序, 而当用户睁开眼睛时,
由于 WmS 中保存了错误的窗口信息, 因此 SurfaceFlinger 就会绘制出错误的界面. 这再次说明, 作为一名程序员, 应该明确的认识到, 获取用户输入消息的仅仅是触摸屏, 而不是触摸屏下面的屏幕, 你当然可以实现触摸底部但是响应顶部区域的效果

### WMS 与 AMS 的关系

#### 从 ActivityStack 类中调用 WmS 接口的地方

| 源自 | 调用者函数                                         | 调用 WmS 中的函数                                                                                                                                                             |
| ---- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A.S  | completeResumeLocked(ActivityRecordnext)           | executeAppTransition()                                                                                                                                                        |
| A.S  | resumeTopActivityLocked(...)                       | setAppVisibility <br> prepareAppTransition()<br> updateOrientationFromAppTokens()<br>executeAppTransition()<br>setAppStartingWindow()<br>addAppToken()<br>validateAppTokens() |
| A.S  | ensureActivitiesVisibleLocked(...)                 | fo r 循 环 中 <br> setAppVisibility(r, true)                                                                                                                                  |
| A.S  | finishActivityLocked( ActivityRecord r, )          | prepareAppTransition(endTask?RANSIT—TASK—CLOSE: TRANSIT—ACTIVITY—CLOSE) prepareAppTransition()                                                                                |
| A.S  | moveTaskToBackLocked(...)                          | moveAppTokensToBottom(moved)<br> validateAppTokens(mHistory)                                                                                                                  |
| A.S  | moveTaskToFrontLocked(..•)                         | prepareAppTransition()<br> moveAppTokensToTop(moved)<br> validateAppTokens(mHistory)                                                                                          |
| A.S  | processStoppingActivitiesLocked(...)               | fo r 循 环 中 <br> setAppVisibility(s, false)                                                                                                                                 |
| A.S  | realStartActivityLocked(...)                       | setAppVisibility(r, true) <br>updateOrientationFromAppTokens()                                                                                                                |
| A.S  | removeActivityFromHistoryLocked( ActivityRecord r) | removeAppToken(r)                                                                                                                                                             |
| A.S  | resetTaskIfNeededLocked(...)                       | setAppGroupId(...) <br>moveAppToken(dstPos, p) <br>validateAppTokens(mHistory)                                                                                                |
| A.S  | startActivityLocked(ActivityRecord,)               | addAppToken() <br>validateAppTokens()<br> prepareAppTransition() <br>setAppStartingWindow() <br>validateAppTokens()                                                           |
| A.S  | stopActivityLocked(ActivityRecordr)                | setAppVisibility(r, false)                                                                                                                                                    |

#### 从 AmS 类中调用 WmS 接口的地方

| 源自 | 调用者函数                                                         | 调用 WmS 中的函数                                        |
| ---- | ------------------------------------------------------------------ | -------------------------------------------------------- |
| AmS  | closeSystemDialogs(String reason)                                  | closeSystemDialogs(reason)                               |
| AmS  | enableScreenAfterBoot()                                            | enableScreenAfterBoot()                                  |
| AmS  | getRequestedOrientation(IBinder token)                             | getAppOrientation(r)                                     |
| AmS  | goingToSleep()                                                     | setEventDispatching(false)                               |
| AmS  | handleAppDiedLocked(...)                                           | while 循 环 中 <br>removeAppToken(r)                     |
| AmS  | isNextTransitionForward()                                          | getPendingAppTransition()                                |
| AmS  | overridePendingTransition(...)                                     | overridePendingAppTransition(...)                        |
| AmS  | setFocusedActivityLocked( ActivityRecord r)                        | setFocusedApp(r, true)                                   |
| AmS  | setRequestedOrientation(IBinder token, int requestedOrientation)   | setAppOrientation(r)<br>updateOrientationFromAppTokens() |
| AmS  | shutdown(int timeout)                                              | setEventDispatching(false)                               |
| AmS  | updateConfiguration(Configuration values)                          | computeNewConfiguration                                  |
| AmS  | updateConfigurationLocked(...) setNewConfiguration(mConfiguration) |
| AmS  | wakingUp                                                           | setEventDispatching(true)                                |

#### 从 ActivityRecord 类中调用 WmS 接口的地方

| 源自 | 调用者函数                              | 调用 WmS 中的函数                 |
| ---- | --------------------------------------- | --------------------------------- |
| A.R  | pauseKeyDispatchingLocked()             | pauseKeyDispatching(this)         |
| A.R  | resumeKeyDispatchingLocked()            | resumeKeyDispatching(this)        |
| A.R  | startFreezingScreenLocked()             | startAppFreezingScreen(this)      |
| A.R  | stopFreezingScreenLocked(boolean force) | stopAppFreezingScreen(this,force) |

![Activity的切换过程---由A到B](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/yiH8TM.jpg)

从 B 到 A 是指, 当前正在运行 B, 然后用户按 Back 键回到 A 的过程

![Activity的切换过程---由B返回到A](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/BVhv5y.jpg)
