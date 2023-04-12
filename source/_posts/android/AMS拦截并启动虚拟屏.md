---
title: AMS拦截并启动虚拟屏到Unity的面板中
urlname: AMS-intercepts-and-starts-the-virtual-screen
date: 2023/04/11
tags:
  - Android
  - Framework
  - DMS
  - AMS
---

## 需求场景

1. Unity 中存在一个面板，这个面板是用来显示 2D 应用的画面的
2. Unity 中的面板是 Unity 创建的，它接收的是一个 TextureID，并创建 Shader 后根据纹理 ID 来渲染
3. 在 android 系统层面，需要拦截所有的 2D 应用并将它们显示到 Unity 的面板中
4. 且这些 2D 应用都是显示到虚拟屏幕中，也就是 VirtualDisplay

先说下这部分我的艰苦历程吧，这个问题其实拖了非常久，因为一开始想到的方案就是多进程共享 Texture 的思路。首先 Unity 应用是一个进程，系统又是一个进程，所以自然而然就想到了多进程共享纹理的思路。一开始直接用固定的纹理 ID 测试后，发现无法实现久没怎么研究了，主要是网上搜到的多进程共享纹理的方案多少有点复杂，就没怎么去看，然后最近才开始重视这个问题，发现其实共享 Surface 就行了。

## Unity 部分

Unity 部分比较简单，使用自带的接口根据 TextureId 来创建 2D 纹理并且创建对应的 Shader 即可

```shader
Shader "Custom/HwAndroid"
{
	Properties
	{
		_MainTex ("Texture", 2D) = "white" {}
	}

	SubShader
	{

		Cull Off
		Pass
		{

			GLSLPROGRAM

				#ifdef VERTEX

				varying vec2 uv;
				void main()
				{
					uv = gl_MultiTexCoord0.xy;
					gl_Position = gl_ModelViewProjectionMatrix * gl_Vertex;
				}

				#endif


				#ifdef FRAGMENT

				#extension GL_OES_EGL_image_external_essl3 : require
				uniform samplerExternalOES _MainTex;
				varying vec2 uv;
				void main()
				{
					gl_FragColor = texture2D(_MainTex, uv);
				}

				#endif

			ENDGLSL
		}
	}

	FallBack "Unlit/Texture"
}

```

Unity 的脚本部分:

```c#
textureId = nativeUnityHolder.Call<int>("createNativeTexture", 1920, 1080, pkg);
texture = Texture2D.CreateExternalTexture(1920, 1080, TextureFormat.RGBA32, false, false, (IntPtr)textureId);
texture.wrapMode = TextureWrapMode.Clamp;
texture.filterMode = FilterMode.Bilinear;
GetComponent<MeshRenderer>().material.shader = Shader.Find("Custom/HwAndroid");
```

## SurfaceTexture

SurfaceTexture 和 Surface 以及 VirtualDisplay 三者之间的关系是：

VirtualDisplay 会接收一个 Surface 参数，会将虚拟屏上的画面画在这个 Surface 上，而 SurfaceTexture 是可以从 Surface 中，将画面转为纹理，而 Unity 则可以根据这个纹理 ID 来渲染。所以流程分以下三步：

1. 在 Server 进程中，根据包名创建虚拟屏，并保存
2. 创建完成后发送消息给 Unity 的 SDK，SDK 根据包名创建对应的 Surface 和 SurfaceTexture，并返回 textureID 给 Unity
3. SDK 将 Surface 返回给 Server 进程，传递给 VirtualDisplay

## VirtualDisplay

每个 2D 应用在 VR 系统中都是无法直接显示的，因为都会直接显示到主屏幕中，就无法根据 VR 眼镜的转动而渲染。

所以一个解决方案是将所有的 2D 应用显示到虚拟屏幕中，创建虚拟屏幕时有一些参数：

```java
VirtualDisplay virtualDisplay = dm.createVirtualDisplay(packageName, width, height, 240, null,
                DisplayManager.VIRTUAL_DISPLAY_FLAG_PUBLIC);
```

其中的 Surface，是我们可以手动操作的地方，我们可以在将 2D 启动到虚拟屏幕后，手动设置这个 Surface

拦截部分的代码是写在 ActivityStater 的 startActivity 中，大约 795 行后，重新设置一遍 checkedOptions

代码如下:

```java
    public ActivityOptions handleAmStart(Context context, RootActivityContainer mRootActivityContainer, Intent intent, ActivityOptions options) {
        String packageName = intent.getComponent().getPackageName();
        //这里需要过滤一下包名
        if (3DUtils.isVrApp(packageName) > 0) {
            return options;
        }
        if (options == null) {
            options = ActivityOptions.makeBasic();
        }
        int virtualDisplayId = VirtualerviceImpl.getInstance().createVirtualDisplay(packageName, 1920, 1080);
        mRootActivityContainer.getActivityDisplayOrCreate(virtualDisplayId);
        options.setLaunchDisplayId(virtualDisplayId);
        return options;
    }
```

其中 getActivityDisplayOrCreate 方法需要手动设置为 public，之所以要调用 getActivityDisplayOrCreate 是因为，当创建虚拟屏后，是异步通知 RootActivityContainer 的
。调用链路如下：

```bash
DMS::handleDisplayDeviceAddedLocked
DMS::addLogicalDisplayLocked
sendDisplayEventLocked(displayId, DisplayManagerGlobal.EVENT_DISPLAY_ADDED);这里是一个异步，发会送`EVENT_DISPLAY_ADDED`
回调 RootActivityContainer 的 onDisplayAdded，通过 getActivityDisplayOrCreate 创建 ActivityDisplay
```

所以，当时想到两个方案，一个是在自己的服务类中接收到 onDisplayAdded 后再处理 Activity 的启动，一个是手动调用 getActivityDisplayOrCreate 来直接创建。显然第一个方案影响了原有功能，所以选择第二个方案。

主动调用 getActivityDisplayOrCreate 的原因在于，当 Activity 在启动过程中，在 startActivityUnchecked 方法下，会调用 mSupervisor.getLaunchParamsController().calculate，里转而到 TaskLaunchParamsModifier 的 calculate，该方法的 getPreferredLaunchDisplay 会去校验一下 DisplayId 对应的 ActivityDisplay

```java
private int getPreferredLaunchDisplay(... ActivityOptions options ...) {
        ......

        int displayId = INVALID_DISPLAY;
        final int optionLaunchId = options != null ? options.getLaunchDisplayId() : INVALID_DISPLAY;
        if (optionLaunchId != INVALID_DISPLAY) {
            displayId = optionLaunchId;
        }

        ......

        if (displayId != INVALID_DISPLAY
                && mSupervisor.mRootActivityContainer.getActivityDisplay(displayId) == null) {
            displayId = currentParams.mPreferredDisplayId;
        }
       ......
    }
```

如果 mRootActivityContainer.getActivityDisplay 获取到的 ActivityDisplay 为 null，displayId 就变为 0 了，所以自然无法启动到指定的虚拟屏上了。
