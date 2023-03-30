---
title: Activity窗口显示和隐藏逻辑分析
urlname: window_showing_hiding
date: 2022/11/11
tags:
  - Window
  - WMS
  - Activity
---

### 启动新的 activity

- Activity.startActivity
- Instrumentation.execStartActivity
- AMS.startActivityAsUser
- ActivityStarter.startActivity

- - 从 caller 获取到调用者的进程信息
  - 创建 ActivityRecord 保存在 r 中
  - 进入 startActivityUnchecked,根据启动模式来处理是否添加新的 task 等

- ActivityStack.startActivityLocked

- - 将要启动的 act 添加到 mTaskHistory 的顶部
  - 如果需要切换新任务时, 在这里做处理, 和窗口相关, 做一些任务切的操作, 包括创建 appWindowToken 和 setAppStartingWindow 等

- 回到 startActivityUnchecked, 会进行 resume 判断, 如果当前要启动的 act 是 resume, 那么直接通过 ensureActivitiesVisibleLocked 来显示, 否则进入 mRootActivityContainer.resumeFocusedStacksTopActivities
- ActivityStack.resumeTopActivityInnerLocked

- - AndroidDisplay.pauseBackStacks
  - ActivityStack.startPausingLocked:如果是立马 pause, 调用 completePauseLocked, 否则发送一个 schedulePauseTimeout（onPause 在 500ms 即 0.5 秒内没有执行完毕的话就会强制关闭 Activity）
  - 会多次进入 resumeTopActivityInnerLocked, 如果 pause 没有结束, 结束的话会调用 completePauseLocked
  - startSpecificActivityLocked（新的进程）

### 非 top Activity 调用 resumeTopActivityUncheckedLocked 分析

假设这个非 top Activity 的 Activity 为 launcher,栈顶的为火狐

launcher 首先需要 resume, 由于非栈顶导致 Activity config changed during resume,直接走 ActivityRecord 的 completeResumeLocked

- completeResumeLocked

- - mStackSupervisor.scheduleIdleTimeoutLocked

- - mStackSupervisor.reportResumedActivityLocked

- - - ensureActivitiesVisibleLocked

- - - - mRootActivityContainer.ensureActivitiesVisible(优先获取栈顶且在 home 之上的 display)

- - - - - display.ensureActivitiesVisible(循环所有的 ActivityStack)

- - - - - stack.ensureActivitiesVisibleLocked(火狐)

- - - - - - activityRecord.makeActiveIfNeeded->shouldResumeActivity(判断当前栈顶的 activity 的状态确定是否需要 resume)

- - - - - - stack.resumeTopActivityInnerLocked(从这里开始就是去 resume 火狐)

- - - - - - stack.pauseBackStacks(在操作之前已经处理了 stack 的顺序, 因此现在的 launcher 在栈中是最顶端, 暂停所有堆栈或 back stacks 中的所有活动, 由于 launcher 在底部, 因此会进行暂停操作)

- - - - - 第二次进入 stack.ensureActivitiesVisibleLocked(launcher)

- - - - - - 走到 makeActiveIfNeeded 的时候发现没有需要 resume 和 pause 的 activity, 直接跳出,至此流程走完

### ensureActivitiesVisibleLocked

ensureActivitiesVisibleLocked 方法用来保证 activity 的可见性, 来确保有可见的 activity

// If the top activity is not fullscreen, then we need to make sure any activities under it are now visible.

从上下到检查哪些 Activity 组件是需要设置为可见的, 哪些 Activity 组件是需要设置为不可见的

从上到下找到第一个全屏显示的 Activity 组件, 并且将该 Activity 组件以及位于该 Activity 组件上面的其它 Activity 组件的可见性设置为 true

home 按键：

com.android.server.policy.PhoneWindowManager.interceptKeyBeforeDispatchingInner:2807

com.android.server.policy.PhoneWindowManager.interceptKeyBeforeDispatching:2666

com.android.server.wm.InputManagerCallback.interceptKeyBeforeDispatching:183

com.android.server.input.InputManagerService.interceptKeyBeforeDispatching:1901

调用时机：

ActivityThread 调用了 handleReusmeActivity 后会发送一个 msg 为 Idle, 在消息机制中空闲的时候进行 activityIdleInternal 处理, 会调用 ensureActivitiesVisibleLocked, 有几个 Activity 就会调用几次, 用来确保一定会有 activity 显示
ActivityStack.resumeTopActivityUncheckedLocked 整个链路下来会调用到
RootActivityContainer.resumeFocusedStacksTopActivities 整个链路下来会调用到
调用路径

如果想要自己的页面不被隐藏, 需要在 ensureActivitiesVisibleLocked 方法中将 behindFullscreenActivity 设置为 false

### ActivityDisplay 的创建

当我们启动一个新的 activity 的时候, 首先会为这个 activity 创建一个 ActivityRecord, 然后在为这个 Activty 分配所属的 TaskRecord, 这个 TaskRecord 又要分配到对应的 ActivityStack, ActivityStack 也要分配到所属的 ActivityDisplay. 这一整套关系确立之后, 我们才能够真正启动这个 Activity

startActivity 中先创建了 ActivityRecord, 在 startActivityUnchecked 中调用 setTaskFromReuseOrCreateNewTask 开始为 ActivityRecord 配置对应的 TaskRecord 和 ActivityStack

```java
private int setTaskFromReuseOrCreateNewTask(TaskRecord taskToAffiliate) {
        if (mRestrictedBgActivity && (mReuseTask == null || !mReuseTask.containsAppUid(mCallingUid))
                && handleBackgroundActivityAbort(mStartActivity)) {
            return START_ABORTED;
        }
				// 创建获取ActivityStack
        mTargetStack = computeStackFocus(mStartActivity, true, mLaunchFlags, mOptions);

        if (mReuseTask == null) {
            final boolean toTop = !mLaunchTaskBehind && !mAvoidMoveToFront;
	          //创建TaskRecord
            final TaskRecord task = mTargetStack.createTaskRecord(
                    mSupervisor.getNextTaskIdForUserLocked(mStartActivity.mUserId),
                    mNewTaskInfo != null ? mNewTaskInfo : mStartActivity.info,
                    mNewTaskIntent != null ? mNewTaskIntent : mIntent, mVoiceSession,
                    mVoiceInteractor, toTop, mStartActivity, mSourceRecord, mOptions);
	          //将ActivityRecord添加到TaskRecord中
            addOrReparentStartingActivity(task, "setTaskFromReuseOrCreateNewTask - mReuseTask");
            updateBounds(mStartActivity.getTaskRecord(), mLaunchParams.mBounds);

            if (DEBUG_TASKS) Slog.v(TAG_TASKS, "Starting new activity " + mStartActivity
                    + " in new task " + mStartActivity.getTaskRecord());
        } else {
            addOrReparentStartingActivity(mReuseTask, "setTaskFromReuseOrCreateNewTask");
        }
				......

        if (mDoResume) {
	          //将当前要显示的ActivityStack移到栈顶
            mTargetStack.moveToFront("reuseOrNewTask");
        }
        return START_SUCCESS;
    }
```
