---
title: Android-WebView手势监听事件的实现
date: 2018-05-21 16:36:24
categories: Android
---

自定义一个 WebView，实现对滑动是否到顶部进行监听。

```java
public class InnerWebView extends WebView {
    private float downX, downY; // 按下时的坐标值
    private float currX, currY; // 移动时的坐标值
    private float moveX;        // 横向移动的长度
    // ...
}
```

<!-- more -->

# 具体实现的代码

```java
public class InnerWebView extends WebView {
    private float downX, downY; // 按下时的坐标值
    private float currX, currY; // 移动时的坐标值
    private float moveX;        // 横向移动的长度

    public InnerWebView(Context context) {
        super(context);
    }

    public InnerWebView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public InnerWebView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
                getParent().getParent().requestDisallowInterceptTouchEvent(true);
                downX = ev.getX();
                downY = ev.getY();
                break;
            case MotionEvent.ACTION_MOVE:
                currX = ev.getX();
                currY = ev.getY();
                moveX = Math.abs(currX - downX);

                // 垂直滑动
                if (Math.abs(currY - downY) > moveX) {
                    float value = getContentHeight() * getResources().getDisplayMetrics().density - getHeight() - getScrollY();
                    Logger.d("value : " + value);
                    // 处于顶部或者无法滚动，并且继续下滑，交出事件（currY-downY >0是下滑, <0则是上滑）
                    if (getScrollY() == 0 && currY - downY > 0) {
                        Logger.d("onTouchEvent ACTION_MOVE 在顶部 下滑 父处理");
                        getParent().getParent().requestDisallowInterceptTouchEvent(false);
                    }
                    // 已到底部且继续上滑时，把事件交出去
                    // getScale被废弃，使用getResources().getDisplayMetrics().density 替代
                    else if((int) (getContentHeight() * getResources().getDisplayMetrics().density) - (getHeight() + getScrollY()) <= 1 && currY - downY < 0){
                        Logger.d("onTouchEvent ACTION_MOVE 在底部 上滑 父处理");
                        getParent().getParent().requestDisallowInterceptTouchEvent(false);
                    }
                }
                // 水平滚动,横向滑动长度大于20像素才认为是水平滑动。
                else if(moveX > 20){
                    // 已在左边且继续右滑时，把事件交出去（currX - downX >0是右滑, <0则是左滑）
                    if (getScrollX() == 0 && currX - downX > 0) {
                        getParent().getParent().requestDisallowInterceptTouchEvent(false);
                    }
                    // 已在右边且继续左滑时，把事件交出去
                    else if(getRight() * getResources().getDisplayMetrics().density - (getWidth() + getScrollX()) <= 1 && currX - downX < 0){
                        getParent().getParent().requestDisallowInterceptTouchEvent(false);
                    }
                }
                break;
            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                getParent().getParent().requestDisallowInterceptTouchEvent(true);
                break;
        }
        return super.onTouchEvent(ev);
    }
}
```