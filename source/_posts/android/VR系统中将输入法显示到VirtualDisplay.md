---
title: VR系统中将输入法显示到VirtualDisplay
urlname: principle-analysis-of-input-method
date: 2023/03/08
tags:
  - android
  - framework
  - IMMS
  - IMS
  - 输入法
  - 软键盘
---

## 需求场景

VR 系统中，Unity 中存在一个面板，需要将系统的 2D 输入法页面显示到这个 3D 面板中，这个面板是一个虚拟屏。

首先，先大致了解一下输入法的流程和术语。

## 缩写解释

- IMS:输入法服务(Input Method Service)，一般是指一个具体的输入法对应的服务。
- IMMS:输入法服务管理器(Input Method Mananger Service)，属于系统进程的一部分，系统中只有一个该服务的实例。
- IMM:输入法管理器(Input Method Manager)，每个客户进程中包含一个该实例。
- IME:(Input Method Engine) , 泛指一个具体的输入法，包括其内部的 IM S 和各种其他 Binder 对象。

## 输入法显示流程

当用户在应用程序中需要输入文本时，Android 系统会自动弹出输入法窗口，该窗口位于应用程序的窗口之上。
输入法窗口会接收用户输入的文本，并将其传递给应用程序。为了实现这一点，输入法窗口使用了一个称为 InputConnection 的接口，该接口将输入的文本传递给应用程序。

在输入法窗口中，用户可以选择不同的输入法类型，例如 QWERTY 键盘、手写输入或语音输入等。输入法窗口会根据用户的选择来展示不同的输入界面。
Android 系统还提供了一个 InputMethodManager(IMM) 类来管理输入法窗口的显示和隐藏。通过该类，应用程序可以请求显示或隐藏输入法窗口，以便更好地管理用户的输入体验。
总的来说，Android 系统层面的输入法窗口原理是在应用程序窗口上方弹出一个悬浮窗口，该窗口可以接收用户输入的文本，并将其传递给应用程序。同时，Android 系统还提供了一些类和接口来管理输入法窗口的显示和隐藏，以便更好地管理用户的输入体验。

## 显示到虚拟屏上

既然输入法的窗口是一个 Dialog，那么我把这个 Dialog 在 addView 的时候人添加到我指定的 DisplayId 不就行了？

所以第一步就是先修改 SoftInputWindow(就是输入法窗口)，先创建一个虚拟屏(注意创建时 flag 要设置为 VIRTUAL_DISPLAY_FLAG_PUBLIC，否则无法跨应用读取到)，然后在 addView 的时候，修改 DisplayId 为固定的 1

现在 WindowManager 类中添加一个字段`softWindowDisplayId`

然后 WindowManagerImpl 代码修改(因为这里的思路是错误的，所以不贴代码，制作思考记录)。

之后在 Dialog 中也创建一个 DisplayId，在 show 方法中 addView 时候传递

## 自动启动输入法

第一种是，用户启动一个新的 Activity 时，或者退出当前 Activity,或者关闭一个系统窗口时，等等，都会导致 WmS 进行窗口切 换，— 旦切换完成，就有一个新的窗口成为交互窗口。此时 WmS 通过 IPC 回调，通知客户端相应的客户端“发生了焦点窗口改变” onWindowFocusChanged()，一旦客户窗口获得该消息后，就开始启动输入法的过程。

第二种是用户主动操作，当点击了输入框之后，调用链是这样的：

```java
android.view.inputmethod.InputMethodManager.focusIn:1794
android.view.View.notifyFocusChangeToInputMethodManager:7951
android.view.View.onFocusChanged:7921
android.widget.TextView.onFocusChanged:10811
android.view.View.handleFocusGainInternal:7597
android.view.View.requestFocusNoSearch:12962
android.view.View.requestFocus:12936
android.view.View.onTouchEvent:15341
android.widget.TextView.onTouchEvent:10870
android.view.View.dispatchTouchEvent:13954
android.view.ViewGroup.dispatchTransformedTouchEvent:3060
android.view.ViewGroup.dispatchTouchEvent:2755
android.view.ViewGroup.dispatchTransformedTouchEvent:3060
com.android.internal.policy.DecorView.superDispatchTouchEvent:466
com.android.internal.policy.PhoneWindow.superDispatchTouchEvent:1849
```

在 TextView.onTouchEvent 中会先调用 super.onTouchEvent(event)，也就是会调用上面的调用链。之后会判断`touchIsFinished`，如果为 true，那么就会调用 viewClicked，走下面的调用链:

```java
android.view.inputmethod.InputMethodManager.viewClicked:2145
android.widget.TextView.viewClicked:12914
android.widget.TextView.onTouchEvent:10915
```

在 viewClick 中，会先判断一个逻辑 `getFallbackInputMethodManagerIfNecessary`

## getFallbackInputMethodManagerIfNecessary

会验证当前的 display 和要显示的 display 他们的 ID 是否相同，（需要验证是否为系统创建？安全性？）
调用 IMM 的进程是哪个？如何确定的？
官方给的注释是

> 正如在 Bug 118341760 中所证明的那样，view.getViewRootImpl().getDisplayId()应该更可靠地确定给定视图正在与哪个显示器交互，而不是 view.getContext().getDisplayId()和 view.getContext().getSystemService()，这些方法可以很容易地被应用程序开发人员（或库作者）弄乱，通过创建不一致的 ContextWrapper 对象，重新分派这些方法到其他上下文，例如 ApplicationContext。

修改记录：

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/1678932738513_BBcc53.png)

原来的逻辑是通过`final int viewDisplayId = view.getContext().getDisplayId();`来进行获取 DisplayId，通过 Context 方法容易在开发者不小心改过之后出现不一致，所以这里的逻辑是要保证 View 所在的 ViewRootImpl 的 DisplayId 和创建 IMM 对象时的 DisplayId 要保持一致。

## DisplayId 的获取来源

IMM 中的 DisplayId 是在创建 IMM 对象时赋值的，他的调用链一般是下面两个情况：

1. 通过 ContextImpl.getSystemService 获取 InputMethodManager 服务时，DisplayId 来源于 Context 的 DisplayId
2. WMS 在 addView 时，会调用到`InputMethodManager`的`ensureDefaultInstanceForDefaultDisplayIfNecessary`方法，用于确保默认显示屏幕上的输入法管理器实例存在，如果当前没有默认显示屏幕上的输入法管理器实例，该方法会通过系统服务获取一个，并将其设置为默认显示屏幕上的实例。

另外一个就是 ViewRootImpl 的 DisplayId，先过一下 addVIew 的流程：

```java
---WindowManagerGlobal.addView()//创建了ViewRootImpl。addView的view是mDecor
----ViewRootImpl.setView()//openSession()创建了Session(IWindowSession的代理类)，view也是mDecor。mDecor传入到ViewRootImpl的mView
-----Session.addToDisplay()//通过Session进入system_server进程
------mService.addWindow()//进入WMS，执行addWindow()添加窗口
```

在第一个 addView 时创建 ViewRootImpl 传入了一个 display，这个 display 同样也是通过 mContext.getDisplay()获取的。

> 由于之前在 addView 中修改的代码有误，导致我误认为 DIsplayId 会不一致而出现问题。其实在 addView 时，通过 mContext 获取的 DisplayId 和 IMM 中的 DisplayId 是一致的，都是虚拟屏的 DisplayId，由于这段分析我觉得需要了解所以进行了保留。

InputMethodManager 的 focusIn 中，会将传递过来的 view(也就是输入框)赋值给 mNextServedView，另一个是判断 ViewRoot 中是否有 CHECK_FOCUS 消息，如果没有则发送一个，这会异步调用 InputMethodManager::checkFocus()函数

该函数中会先检查 served 视图是否变化。比如，如果用户在同一个 TextView 上连续点击时 ，也会执行到 checkFocus()函 数 ，但是由于视图没变化，因此该函数会立即返回 ， 检查的条件是对比 mServedView 和 mNextServedView。当第一次调用 checkFocus()函数时，这两个变量值不相同，而执行后，mServedView 被赋值为 mNextServedView.

当客户端要显示输入法窗口时，会调用 IMM 的 showSoftInput()函数，该函数中则会调用 IMMS 的 同名函数。在 IMMS 中釆用的是异步机制，即先发送一个 MSG_SHOW_SOFT_INPUT 的消息，发送消 息时，调用了 IMMS 的内部函数 executeOrSendMessage()，该函数的参数使用 mCaller.obtainMessageIOO() 函数进行创建。mCaller 是一个 HandlerCaller 类。接下来就会执行到当前 IMS 中的同名函数 showSoftInput()中

### getDisplayContentOrCreate

查看 dumps window 的时候，发现 displayid 并不是为 1，查看到 dumps 的代码是在 WindowState 的 dump 方法中

```java
    @Override
    public int getDisplayId() {
        final DisplayContent displayContent = getDisplayContent();
        if (displayContent == null) {
            return Display.INVALID_DISPLAY;
        }
        return displayContent.getDisplayId();
    }
```

这个 DisplayContent 是在 WMS 的 addWindow 时，通过 getDisplayContentOrCreate 获取

getDisplayContentOrCreate 中的逻辑是这样的：

```java
    private DisplayContent getDisplayContentOrCreate(int displayId, IBinder token) {
        if (token != null) {
            final WindowToken wToken = mRoot.getWindowToken(token);
            if (wToken != null) {
                return wToken.getDisplayContent();
            }
        }
        DisplayContent displayContent = mRoot.getDisplayContent(displayId);
        if (displayContent == null) {
            final Display display = mDisplayManager.getDisplay(displayId);
            if (display != null) {
                displayContent = mRoot.createDisplayContent(display, null /* controller */);
            }
        }
        return displayContent;
    }
```

这个函数的作用就是确定 view 要显示在哪个屏幕上。

那么这个 token 从何而来？

## token

**启动输入法是在客户窗口变成当前交互窗口时开始执行的**，所以当我们启动一个 app 到虚拟屏时，就会启动输入法。这里说的启动仅仅是启动输入法服务，即 IMS，并非会显示输入法窗口。

如果是输入法窗口 ， 则 attr.token 必须存在，并且其类型必须是 INPUT_METHOD，对于应用程序而言，没有权限创建输入法窗口，该窗口是在 IMMS 中创建的，创建之前，它会先创建一个 WindowToken 对象，创建的链路如下：

1. IMMS::startInputUncheckedLocked
2. IMMS::bindCurrentInputMethodServiceLocked
3. IMMS::mIWindowManager.addWindowToken
4. IMMS::onServiceConnected
5. start IMS

可以看到，WindowToken 是在上面的第三个步骤添加的，并且和 DisplayContent 进行了绑定。

而这里的创建的 WindowToken 和 getDisplayContentOrCreate 中的参数 token 就是同一个。在 addWindowToken 的流程中，会根据传递的 DisplayId 来创建对应的 WindowToken。

但是发现，当我在虚拟屏上起 App 时，addWindowToken 的 DisplayId 总是为 0，也就是默认屏幕。addWindowToken 的 DisplayId 参数来源是 IMMS 中的 computeImeDisplayIdForTarget，用来找哪个屏幕应该显示输入法。

总结一下，首先在 WMS 中会通过 getDisplayContentOrCreate 来找到 DisplayContent，DisplayContent 代表了哪一个屏幕，后续获取 WindowToken、创建 WIndowState 以及 addWindow 等操作都是基于这个 DisplayContent,所以，包括一开始在 addView 中修改的代码都可以删除，只需要在 computeImeDisplayIdForTarget 中返回自己创建的虚拟屏 ID 即可。

## 窗口大小

显示之后，会出现大小不对应的情况，如下图：

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/1679041574968_DWGdrz.png)

首先贴出在 Unity 端的代码:

```c#
    void Start()
    {
        textureId = nativeUnityHolder.Call<int>("showSoft", 1920, 1080, "soft");
        texture = Texture2D.CreateExternalTexture(1920, 1080, TextureFormat.RGBA32, false, false, (IntPtr)textureId);
        texture.wrapMode = TextureWrapMode.Clamp;
        texture.filterMode = FilterMode.Bilinear;
        GetComponent<MeshRenderer>().material.shader = Shader.Find("Android");
    }
```

这段代码的意思就是创建了一个宽高为 1920\*1080 的 texture 来接收 Android 给过来的画面数据，看起来毫无破绽。

其实问题就在这个 1920\*1080。

首先，一开始考虑出现这个情况，原因大概率就是键盘 View 的大小并不是跟着父布局或者是给定的大小来的。

在系统的 KeyboardView 的 onMeasure 中

```java
    @Override
    protected void onMeasure(final int widthMeasureSpec, final int heightMeasureSpec) {
        final Keyboard keyboard = getKeyboard();
        if (keyboard == null) {
            super.onMeasure(widthMeasureSpec, heightMeasureSpec);
            return;
        }
        // The main keyboard expands to the entire this {@link KeyboardView}.
        final int width = keyboard.mOccupiedWidth + getPaddingLeft() + getPaddingRight();
        final int height = keyboard.mOccupiedHeight + getPaddingTop() + getPaddingBottom();
        setMeasuredDimension(width, height);
    }
```

mOccupiedWidth 就是键盘的整个宽度,mOccupiedWidth 的值,默认是这样获取的:

```java
    public static int getDefaultKeyboardWidth(final Resources res) {
        final DisplayMetrics dm = res.getDisplayMetrics();
        return dm.widthPixels;
    }
```

可以看到是通过 Resources 来获取到 DisplayMetrics 的宽度.
