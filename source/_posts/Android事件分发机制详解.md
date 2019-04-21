---
title: Android事件分发机制详解
tags:
  - Android
categories:
  - Android事件分发
description: 
comments: true
date: 2018-08-25 12:22:04

typora-root-url: ./Android事件分发机制详解
typora-copy-images-to: ./Android事件分发机制详解
---

## 什么是View事件分发

这里的事件是指手指触屏到屏幕（Down）到手指离开屏幕（Up，Cancel）以及中间一系列（Move），这一连串的事件也叫事件序列。在Android开发中了解事件分发机制很重要，自定义View以及处理滑动冲突的时候都需要用到。

虽然事件分发时包含了从down到up过程的事件序列，但是这里更倾向去将事件分发定义为事件Down的分发，因为Down事件会决定后续所有事件的走向。下面我们从实际的例子中来看View的事件时怎么分发的。

先来一张简图：
![WX20180825-170921](/WX20180825-170921.png)
<!-- more -->

## 事件分发过程

写一个层级大概如下的视图：

![1.png](/1.png)

重写各个层级的Activity，ViewGroup，View的几个关键事件传递的方法。
1. boolean dispatchTouchEvent(MotionEvent ev)
2. boolean onInterceptTouchEvent(MotionEvent ev)
3. boolean onTouchEvent(MotionEvent event)

注意: 
1. 这3个方法都用都有相同的输入和输出，输入为事件类型，输出为是否消费事件
2. Activity 和 View 没有onInterceptTouchEvent，只有ViewGroup才有。



```
public class MainActivity extends AppCompatActivity {
    // ...

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        Log.e(TAG, "dispatchTouchEvent: " + ev.getAction());
        return super.dispatchTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        Log.e(TAG, "onTouchEvent: " + event.getAction());
        return super.onTouchEvent(event);
    }
}
```
```
public class FrameLayout1 extends FrameLayout {
    // ...

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        Log.i(TAG, "dispatchTouchEvent: " + ev.getAction());
        return super.dispatchTouchEvent(ev);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        Log.i(TAG, "onInterceptTouchEvent: " + ev.getAction());
        return super.dispatchTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        Log.i(TAG, "onTouchEvent: " + ev.getAction());
        return super.dispatchTouchEvent(ev);
    }
}
```
LinerLayout 同上FrameLayout ......

```
public class MyTextView extends android.support.v7.widget.AppCompatTextView {
    // ...

    @Override
    public boolean dispatchTouchEvent(MotionEvent event) {
        Log.d(TAG, "dispatchTouchEvent: " + event.getAction());
        return super.dispatchTouchEvent(event);;
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        Log.d(TAG, "onTouchEvent: " + event.getAction());
        return super.dispatchTouchEvent(event);;
    }
}
```
###  1. **目前的代码没有拦截任何事件，都是直接调用 supper.xxx() 传递事件** 然后在屏幕上做一个滑动动作，拿到如下事件：

PS: MotionEvent.getAction() 事件是 int 常量
0 -> ACTION_DOWN
1 -> ACTION_UP
2 -> ACTION_MOVE
3 -> ACTION_CANCEL 
......

```
E/MainActivity: dispatchTouchEvent: 0
I/FrameLayout1: dispatchTouchEvent: 0
I/FrameLayout1: onInterceptTouchEvent: 0
D/Linerlayout1: dispatchTouchEvent: 0
D/Linerlayout1: onInterceptTouchEvent: 0
D/MyTextView: dispatchTouchEvent: 0
D/MyTextView: onTouchEvent: 0
D/Linerlayout1: onTouchEvent: 0
I/FrameLayout1: onTouchEvent: 0
E/MainActivity: onTouchEvent: 0
E/MainActivity: dispatchTouchEvent: 2
E/MainActivity: onTouchEvent: 2
E/MainActivity: dispatchTouchEvent: 2
E/MainActivity: onTouchEvent: 2
E/MainActivity: dispatchTouchEvent: 2
E/MainActivity: onTouchEvent: 2
E/MainActivity: dispatchTouchEvent: 1
E/MainActivity: onTouchEvent: 1
```
默认传递过程如下：
Down事件：从Activity的dispatch -> Viewgroup的dispatch, intercept -> View的 dispatch, ontouch -> 父ViewGroup的 ontouch -> Activity的 ontouch
Move事件：全部交给Activity
Up事件：  全部交给Activity

### 2. 在Framelayout的dispatchTouchEvent 中的 Down 事件中拦截返回 true，或 false

```
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    Log.i(TAG, "dispatchTouchEvent: " + ev.getAction());
    switch (ev.getAction()) {
        case MotionEvent.ACTION_DOWN:
            return true; // false
        case MotionEvent.ACTION_MOVE:
            break;
        case MotionEvent.ACTION_UP:
            break;
    }
    return super.dispatchTouchEvent(ev);
}
```
**返回true**
```
E/MainActivity: dispatchTouchEvent: 0
I/FrameLayout1: dispatchTouchEvent: 0
E/MainActivity: dispatchTouchEvent: 2
I/FrameLayout1: dispatchTouchEvent: 2
I/FrameLayout1: onTouchEvent: 2
E/MainActivity: onTouchEvent: 2
E/MainActivity: dispatchTouchEvent: 2
I/FrameLayout1: dispatchTouchEvent: 2
I/FrameLayout1: onTouchEvent: 2
E/MainActivity: onTouchEvent: 2
E/MainActivity: dispatchTouchEvent: 1
I/FrameLayout1: dispatchTouchEvent: 1
I/FrameLayout1: onTouchEvent: 1
E/MainActivity: onTouchEvent: 1
```
**返回false**
```
E/MainActivity: dispatchTouchEvent: 0
I/FrameLayout1: dispatchTouchEvent: 0
E/MainActivity: onTouchEvent: 0
E/MainActivity: dispatchTouchEvent: 2
E/MainActivity: onTouchEvent: 2
E/MainActivity: dispatchTouchEvent: 2
E/MainActivity: onTouchEvent: 2
E/MainActivity: dispatchTouchEvent: 1
E/MainActivity: onTouchEvent: 1
```
得出结论：
如果在dispatch中并且事件是down，返回true和false都会导致事件不向子View传递（因为没有调用super.dispatchXXX())，返回true和false的区别在于：
1. 返回true表示此view消耗了事件，下一次move事件依然会传递到这里，并且直接传递给onTouchEvent（不在调用onIntercept）
2. 如果返回false，由于Framelayout上一层就是Activity，所以之后的事件都不会再传递下来。

### 3. 在Framelayout的onIntercept中且事件未Down时返回true或false

```
public boolean onInterceptTouchEvent(MotionEvent ev) {
    Log.i(TAG, "onInterceptTouchEvent: " + ev.getAction());
    switch (ev.getAction()) {
        case MotionEvent.ACTION_DOWN:
            return true; // false
        case MotionEvent.ACTION_MOVE:
            break;
        case MotionEvent.ACTION_UP:
            break;
    }
    return super.onInterceptTouchEvent(ev);
}
```

**返回true**
```
E/MainActivity: dispatchTouchEvent: 0
I/FrameLayout1: dispatchTouchEvent: 0
I/FrameLayout1: onInterceptTouchEvent: 0
I/FrameLayout1: onTouchEvent: 0
E/MainActivity: onTouchEvent: 0
E/MainActivity: dispatchTouchEvent: 2
E/MainActivity: onTouchEvent: 2
E/MainActivity: dispatchTouchEvent: 2
E/MainActivity: onTouchEvent: 2
```

**返回false**
```
E/MainActivity: dispatchTouchEvent: 0
I/FrameLayout1: dispatchTouchEvent: 0
I/FrameLayout1: onInterceptTouchEvent: 0
D/Linerlayout1: dispatchTouchEvent: 0
D/Linerlayout1: onInterceptTouchEvent: 0
D/Linerlayout1: onTouchEvent: 0
I/FrameLayout1: onTouchEvent: 0
E/MainActivity: onTouchEvent: 0
E/MainActivity: dispatchTouchEvent: 2
E/MainActivity: onTouchEvent: 2
E/MainActivity: dispatchTouchEvent: 2
E/MainActivity: onTouchEvent: 2
```

当事件为Down时，如果onInterceptTouchEvent() 

1. 返回true，表示拦截，直接调用自己的onTouchEvent，事件不再向下传递
2. 返回false，不拦截（默认），继续传递事件到子view，流程和调用super.onInterceptXXX一样



### 4. 在LinearLayout的onTouchEvent中返回true或false

```
public boolean onTouchEvent(MotionEvent ev) {
    Log.i(TAG, "onTouchEvent: " + ev.getAction());
    switch (ev.getAction()) {
        case MotionEvent.ACTION_DOWN:
            return true; // false
        case MotionEvent.ACTION_MOVE:
            break;
        case MotionEvent.ACTION_UP:
            break;
    }
    return super.onInterceptTouchEvent(ev);
}
```

**返回true：**

```
E/MainActivity: dispatchTouchEvent: 0
I/FrameLayout1: dispatchTouchEvent: 0
I/FrameLayout1: onInterceptTouchEvent: 0
D/Linerlayout1: dispatchTouchEvent: 0
D/Linerlayout1: onInterceptTouchEvent: 0
D/MyTextView: dispatchTouchEvent: 0
D/MyTextView: onTouchEvent: 0
D/Linerlayout1: onTouchEvent: 0
E/MainActivity: dispatchTouchEvent: 2
I/FrameLayout1: dispatchTouchEvent: 2
I/FrameLayout1: onInterceptTouchEvent: 2
D/Linerlayout1: dispatchTouchEvent: 2
--------------未调用自己的 onInterceptTouchEvent
D/Linerlayout1: onTouchEvent: 2
--------------未调用父View的 onTouchEvent
E/MainActivity: onTouchEvent: 2
E/MainActivity: dispatchTouchEvent: 1
I/FrameLayout1: dispatchTouchEvent: 1
I/FrameLayout1: onInterceptTouchEvent: 1
D/Linerlayout1: dispatchTouchEvent: 1
D/Linerlayout1: onTouchEvent: 1
E/MainActivity: onTouchEvent: 1

```

**返回false：**

```
E/MainActivity: dispatchTouchEvent: 0
I/FrameLayout1: dispatchTouchEvent: 0
I/FrameLayout1: onInterceptTouchEvent: 0
D/Linerlayout1: dispatchTouchEvent: 0
D/Linerlayout1: onInterceptTouchEvent: 0
D/MyTextView: dispatchTouchEvent: 0
D/MyTextView: onTouchEvent: 0
D/Linerlayout1: onTouchEvent: 0
I/FrameLayout1: onTouchEvent: 0
E/MainActivity: onTouchEvent: 0
E/MainActivity: dispatchTouchEvent: 2
E/MainActivity: onTouchEvent: 2
E/MainActivity: dispatchTouchEvent: 2
E/MainActivity: onTouchEvent: 2
E/MainActivity: dispatchTouchEvent: 1
E/MainActivity: onTouchEvent: 1

```

当事件为Down时，如果onTouchEvent：

**返回true：**表示事件被此view消耗，之后所有move，cancel事件都会按照从上到下的路径传递到此view。但是需要注意，这个过程中**自己的onIntercept没有被调用**，并且**自己的onTouchEvent之后直接调用Activity的onTouchevent，没有走Framelayout的onTouch**。

**返回false：**表示不消费事件，onTouch继续上传给父View，直到Activity，之后所有event都由Activity消费。

## 小结

对于事件Down来说非常重要，因为它决定了后续事件的流向。下图只是对于Down事件的一些case，还有很多case图上没有表现，可参考上面集中4种情况的log分析。

![WX20180825-170921](/WX20180825-170921.png)

对于Down事件决定事件的流向可以这样理解，如果事件为Down时候，如果在dispatchTouchEvent 和 onTouchevent中只要返回true，那么该View会收到后续的所有事件。**因为事件传递过程中记录了一个mFirstTouchTarget的对象，表示消费事件的view，如果确定了这个target那么事件会直接交给target处理。**

如果Down事件按照默认流程U型传递一圈发现没有消耗Down事件，那么后续事件也没有必要再向下传递，Activity直接消耗完即可。

**PS: 还有一个需要注意的地方，如果一个View的clickable或者longclickable的属性被设置为true，那么在它的onTouchEvent中会默认返回true。比如Button。这会导致默认情况它就会消耗事件**

