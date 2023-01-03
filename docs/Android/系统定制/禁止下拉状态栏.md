上锁屏界面状态栏要禁止下拉请按如下方案修改：

NotificationPanelView.java(alps/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone)中的两个方法

（1）

```java
private void setQsExpanded(boolean expanded) {
        //begin 添加下面四行
        if(mKeyguardShowing)                     
        {                                                   
              return;                                     
        }                                                  
        //end
        boolean changed = mQsExpanded != expanded;
        if (changed) {
            mQsExpanded = expanded;
            updateQsState();
            requestPanelHeightUpdate();
            mNotificationStackScroller.setInterceptDelegateEnabled(expanded);
            mStatusBar.setQsExpanded(expanded);
        }
    }
```

(2)

```java
    private boolean shouldQuickSettingsIntercept(float x, float y, float yDiff) {
        if (!mQsExpansionEnabled) {
            return false;
        }
        //begin 将下面第一行替换成第二行
        View header = mKeyguardShowing ? mKeyguardStatusBar : mHeader;       
        View header = mHeader;
       //end                                                                     
        boolean onHeader = x >= header.getLeft() && x <= header.getRight()
                && y >= header.getTop() && y <= header.getBottom();
        if (mQsExpanded) {
            return onHeader || (mScrollView.isScrolledToBottom() && yDiff < 0) && isInQsArea(x, y);
        } else {
            return onHeader;
        }
    }
 
```

(3)

```java
private boolean onTouchEvent()
{
...
        if (!mTwoFingerQsExpand && mQsTracking) {
            //begin  添加下面红色的两行
            if(!mKeyguardShowing){                                   
                onQsTouch(event);
                if (!mConflictingQsExpansionGesture) {
                    return true;
                }
            }
             //end　　　　　　　　　　　　　　　　　　　　　　
        }
...
}

```
