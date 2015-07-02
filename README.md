#Android键盘面板冲突 布局闪动处理方案

> 之前有写过一篇核心思想: [Switching between the panel and the keyboard in Wechat](http://blog.dreamtobe.cn/2015/02/07/Switching-between-the-panel-and-the-keyboard/)

我们可以看到微信中的 从键盘与微信的切换是无缝的，而且是无闪动的，这种基础体验是符合预期的。

但是实际中，简单的 键盘与面板切换 是会有闪动，问题的。今天我们就实践分析与解决这个问题。

<!--more-->
## 最终效果对比:

![](https://raw.githubusercontent.com/Jacksgong/JKeybordPanelSwitch/master/img/resolve_mv.gif)![](https://raw.githubusercontent.com/Jacksgong/JKeybordPanelSwitch/master/img/unresolve_mv.gif)


## I. 准备

> 以下建立在`android:windowSoftInputMode`带有`adjustResize`的基础上。

> 如图，为了方便分析，我们分出3个View:

![](https://raw.githubusercontent.com/Jacksgong/JKeybordPanelSwitch/master/img/WeChat_1435588322.png)

- `CustomRootView`: 除去statusBar与ActionBar(ToolBar...balabala)
- `FootRootView`: 整个底部（包括输入框与底部面板在内的整个View）
- `PanelView`: 面板View

> 整个处理过程，其实需要分为两块处理:

1. 从`PanelView`切换到`Keybord`
 
**现象:** 由于显示`Keybord`时直接`PanelView#setVisibility(View.GONE)`，导致会出现整个`FooterRootView`到底部然后又被键盘顶起。

**符合预期的应该:** 直接被键盘顶起，不需要到底部再顶起。

2. 从`Keybord`切换到`PanelView`

**现象:** 由于隐藏`Keybord`时，直接`PanelView#setVisibility(View.VISIBLE)`，导致会出现整个`FootRootView`先被顶到键盘上面，然后再随着键盘的动画，下到底部。

**符合预期的应该:** 随着键盘收下直接切换到底部，而配有被键盘顶起的闪动。

## II. 处理

### 1. 从`PanelView`切换到`Keybord`

#### 原理 

屏蔽由于`PanelView#setVisibility(View.GONE)`导致，到底部的那一帧。

#### 方法: 

> 不直接调用`PanelView#setVisibility(View.GONE)`来隐藏`PanelView`，由于调用`setVisiblility`会促发`requestLayout`，将直接导致这帧被绘制。而是设置一个标志位，在由于键盘显示导致`PanelView`重新mearsure调用`onMeasure`的时候处理。

如代码:

```
@Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		if (isHide) {
			setVisibility(View.GONE);
			widthMeasureSpec = MeasureSpec.makeMeasureSpec(0, MeasureSpec.EXACTLY);
			heightMeasureSpec = MeasureSpec.makeMeasureSpec(0, MeasureSpec.EXACTLY);
		}
		super.onMeasure(widthMeasureSpec, heightMeasureSpec);
	}
```

### 2. 从`Keybord`切换到`PanelView`

#### 原理

在真正由`Keybord`导致布局**真正**将要变化的时候，才给`PanelView`有效高度。否则直接给0高度。（**注意**，所有的判断处理要在`Super.onMeasure`之前完成判断）

#### 方法:

> 通过`CustomRootView`高度的变化，来保证在`Super.onMeasure`之前获得**真正**的由于键盘导致布局将要变化，然后告知`PanelView`，让其在`Super.onMeasure`之前给到有效高度。

#### 需要注意:

> 1) 在`adjustResize`模式下，键盘弹起会导致`CustomRootView`的高度变小，键盘收回会导致`CustomRootView`的高度变大。因此可以通过这个机制获知真正的`PanelView`将要变化的时机。


> 2) 由于到了`onLayout`，clipRect的大小已经确定了，又要避免不多次调用`onMeasure`因此要在`Super.onMeasure`之前 


> 3) 由于键盘收回的时候，会触发多次`measure`，如果 不判断真正的由于键盘收回导致布局将要变化，就直接给有效高度，依然会有闪动的情况。

> 4) 从`Keybord`切换到`PanelView`导致的布局冲突，只有在`Keybord`正在显示的时候，才有可能触发，因此我们需要过滤掉`Keybord`未显示的情况。

代码:

`CustomRootView`

```
private int mOldHeight = -1;

@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

    // 由当前布局被键盘挤压，获知，由于键盘的活动，导致布局将要发生变化。
    do {
        final int width = MeasureSpec.getSize(widthMeasureSpec);
        final int height = MeasureSpec.getSize(heightMeasureSpec);

        Log.d(TAG, "onMeasure, width: " + width + " height: " + height);
        if (height < 0) {
            break;
        }

        if (mOldHeight < 0) {
            mOldHeight = height;
            break;
        }

        final int offset = mOldHeight - height;
        mOldHeight = height;

        if (offset >= 0) {
            //键盘弹起 (offset > 0，高度变小)
            Log.d(TAG, "" + offset + " >= 0 break;");
            break;
        }

        final PanelLayout bottom = getPanelLayout(this);

        if (bottom == null) {
            Log.d(TAG, "bottom == null break;");
            break;
        }

        // 检测到真正的 由于键盘收起触发了本次的布局变化
        bottom.setIsNeedHeight(true);

    } while (false);

    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

}

private PanelLayout mPanelLayout;

private PanelLayout getPanelLayout(final View view) {
    if (mPanelLayout != null) {
        return mPanelLayout;
    }

    if (view instanceof PanelLayout) {
        mPanelLayout = (PanelLayout) view;
        return mPanelLayout;
    }

    if (view instanceof ViewGroup) {
        for (int i = 0; i < ((ViewGroup) view).getChildCount(); i++) {
            PanelLayout v = getPanelLayout(((ViewGroup) view).getChildAt(i));
            if (v != null) {
                mPanelLayout = v;
                return mPanelLayout;
            }
        }
    }

    return null;

}

private int maxBottom = 0;

@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    super.onLayout(changed, l, t, r, b);

    if (b >= maxBottom && maxBottom != 0) {
        // 在底部，键盘隐藏状态
        Log.d(TAG, "keybor hiding");
        getPanelLayout(this).setIsKeybordShowing(false);
    } else if(maxBottom != 0){
        Log.d(TAG, "keybor showing");
        getPanelLayout(this).setIsKeybordShowing(true);
    }

    if (maxBottom < b) {
        maxBottom = b;
    }

}
```

`PanelView`

```
@Override
public void setVisibility(int visibility) {
    if (visibility == getVisibility()) {
        return;
    }

    if (mIsKeybordShowing) {
        //只有在键盘显示的时候才需要处理 keybord -> panel切换的布局冲突问题
        setIsNeedHeight(false);

        ViewGroup.LayoutParams l = getLayoutParams();
        l.height = 0;
        setLayoutParams(l);
    }
    super.setVisibility(visibility);

}

@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

    if (getVisibility() == View.VISIBLE && mIsNeedHeight) {
        // 真正需要高度的时候（是否需要高度由是否是键盘触发布局真正要发生变化时告知 & visible）。
        ViewGroup.LayoutParams l = getLayoutParams();
        setVisibility(View.VISIBLE);
        l.height = mHeight;
        heightMeasureSpec = MeasureSpec.makeMeasureSpec(mHeight, MeasureSpec.EXACTLY);
    }

    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
}


private boolean mIsKeybordShowing = false;
public void setIsKeybordShowing(final boolean isKeybordShowing) {
    this.mIsKeybordShowing = isKeybordShowing;
}

private boolean mIsNeedHeight = true;
public void setIsNeedHeight(final boolean isNeedheight) {
    this.mIsNeedHeight = isNeedheight;
}
```

## III. License

```
Copyright 2015 Jacks Blog.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```