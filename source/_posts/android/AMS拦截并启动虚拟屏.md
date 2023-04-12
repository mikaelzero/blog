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

createNativeTexture 是 native 的一个 SDK 中的方法，主要作用就是创建 Surface 以及 SurfaceTexture，大致如下：

```java
    private int createGLTexture(int width, int height) {
        int[] textures = new int[1];
        GLES30.glGenTextures(1, textures, 0);
        int textureId = textures[0];
        GLES30.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, textureId);
        GLES30.glTexParameteri(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, GLES30.GL_TEXTURE_MIN_FILTER, GLES30.GL_LINEAR_MIPMAP_LINEAR);
        GLES30.glTexParameteri(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, GLES30.GL_TEXTURE_MAG_FILTER, GLES30.GL_LINEAR_MIPMAP_LINEAR);
        GLES30.glTexParameteri(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, GLES30.GL_TEXTURE_WRAP_S, GLES30.GL_CLAMP_TO_EDGE);
        GLES30.glTexParameteri(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, GLES30.GL_TEXTURE_WRAP_T, GLES30.GL_CLAMP_TO_EDGE);
        GLES30.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, 0);

        GLES30.glGetError();
        return textureId;
    }

    private void attachGLTexture(int textureId) {
        mTextureId = textureId;
        mSurfaceTexture = new SurfaceTexture(mTextureId);
        mSurfaceTexture.setDefaultBufferSize(mSurfaceFrame.width(), mSurfaceFrame.height());
        mSurfaceTexture.setOnFrameAvailableListener(this);

        mSurface = new Surface(mSurfaceTexture);
    }

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

其中的 Surface，是我们可以操作的地方，我们可以在将 2D 启动到虚拟屏幕后，手动设置这个 Surface,这个 Surface 就是上面 Unity 部分创建的 Surface，这样 Unity 就能够获取到虚拟屏幕的画面。

拦截部分的代码是写在 ActivityStater 的 startActivity 中，大约 795 行后，在创建 ActivityRecord 之前，重新设置一遍 checkedOptions。

代码如下:

```java
    public ActivityOptions handleStartActivity(Context context, RootActivityContainer mRootActivityContainer, Intent intent, ActivityOptions options) {
        String packageName = intent.getComponent().getPackageName();
        //这里需要过滤一下包名，不仅仅是3D，其他的系统应用也要过滤
        if (3DUtils.isVrApp(packageName) > 0) {
            return options;
        }
        if (options == null) {
            options = ActivityOptions.makeBasic();
        }
        int virtualDisplayId = VirtualServiceImpl.getInstance().createVirtualDisplay(packageName, 1920, 1080);
        options.setLaunchDisplayId(virtualDisplayId);
        return options;
    }
```

修改后发现，在 handleStartActivity 方法中的 virtualDisplayId 为正确的值，也就是非 0，但是最终在 dumps activitys 时，2D 总是显示在主屏幕上，说明在拦截了之后，肯定还有个地方修改了 displayId。

当 Activity 在启动过程中，在 startActivityUnchecked 方法下，会调用 mSupervisor.getLaunchParamsController().calculate 然后转到 TaskLaunchParamsModifier 的 calculate，该方法的 getPreferredLaunchDisplay 会去校验一下 DisplayId 对应的 ActivityDisplay

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

那么为什么明明启动了虚拟屏，却没有找到呢？说明肯定有个地方是异步的，AOSP 中到处都是 Handler 通信，所以创建完了虚拟屏后，肯定做了异步处理。

果不其然，当调用 createVirtualDisplay 创建虚拟屏后，一路走到 DMS 的 handleDisplayDeviceAddedLocked 方法，该方法会调用 addLogicalDisplayLocked，之后会发送一个 Handler 的 Message

```java
sendDisplayEventLocked(displayId, DisplayManagerGlobal.EVENT_DISPLAY_ADDED);
```

该方法会通过 Handler 发送一条消息，最终在 RootActivityContainer 中会进行接收，回调 RootActivityContainer 的 onDisplayAdded 后，通过 getActivityDisplayOrCreate 创建 ActivityDisplay。

既然如此，那就必须等到 ActivityDisplay 创建完毕才能启动我们的 Activity。

一开始想到的是在自己的服务类中接收到 onDisplayAdded 后再处理 Activity 的启动，当时这个方案改动太大且影响了原有 Activity 启动逻辑，所以废弃。

既然最终是要在 RootActivityContainer 中通过 getActivityDisplayOrCreate 创建 ActivityDisplay，那我手动调用 getActivityDisplayOrCreate 不就行了？

所以这部分的代码改动如下：

```java
    public ActivityOptions handleStartActivity(Context context, RootActivityContainer mRootActivityContainer, Intent intent, ActivityOptions options) {
        ......
        int virtualDisplayId = VirtualServiceImpl.getInstance().createVirtualDisplay(packageName, 1920, 1080);
        //getActivityDisplayOrCreate 方法需要手动设置为 public
        mRootActivityContainer.getActivityDisplayOrCreate(virtualDisplayId);
        options.setLaunchDisplayId(virtualDisplayId);
        return options;
    }
```

## 总结

总结几个注意点：

1. 自己写的服务需要在 Server 进程，这样才能和 Server 进程中的 AMS 通信，无需用 Binder 来处理
2. 写完后记得 make update api
3. Unity 进程传递 Surface 到 Server 进程需要用到 Binder
4. 虚拟屏的 flag 需要为 VIRTUAL_DISPLAY_FLAG_PUBLIC
5. 启动拦截时，注意过滤的包名，比如一开始启动的 FallbackHome 的 Activity 或者是 SystemUI 都需要过滤不处理