---
title: LiveData源码解析
tags:
  - Android
categories:
  - Android源码解析系列
comments: true
date: 2019-03-24 19:13:34
typora-copy-images-to: ./LiveData源码解析
---

## 介绍

LiveData是Android Jetpack Conponent中一个重要的组件，它能将数据与Android生命周期组件(Activity/Fragment/Service等，注：下文讲解实例中直接将生命周期组件替换成Activity，其他组件大同小异)进行有机的绑定，当数据变化时只通知active的组件，当组件销毁时自动解除关系避免内存泄漏，避免了我们在生命周期中写很长的代码来拉去数据更新UI，将数据变化与UI Control分离开，非常好用。下文我们将分析LiveData源码是怎么做到的，文中例子代码直接来自于Android Developer官方文档示例代码。

地址：<https://developer.android.com/topic/libraries/architecture/livedata>

来一张简图：

![](./image-20190324221458032.png)

<!-- more -->

## LiveData代码示例

1. 创建ViewModel类，里面存放LiveData对象

```java
public class NameViewModel extends ViewModel {

// Create a LiveData with a String
private MutableLiveData<String> currentName;

    public MutableLiveData<String> getCurrentName() {
        if (currentName == null) {
            currentName = new MutableLiveData<String>();
        }
        return currentName;
    }

// Rest of the ViewModel...
}
```

2. 通过Observer将LiveData与生命周期组件绑定，在OnChanged回调中修改UI

```java
public class NameActivity extends AppCompatActivity {

    private NameViewModel model;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // Other code to setup the activity...

        // Get the ViewModel.
        model = ViewModelProviders.of(this).get(NameViewModel.class);

        // Create the observer which updates the UI.
        final Observer<String> nameObserver = new Observer<String>() {
            @Override
            public void onChanged(@Nullable final String newName) {
                // Update the UI, in this case, a TextView.
                nameTextView.setText(newName);
            }
        };

        // Observe the LiveData, passing in this activity as the LifecycleOwner and the observer.
        model.getCurrentName().observe(this, nameObserver);
    }
}
```

3. 使用LiveData通知数据变化，LiveData更新数据有两个方法setValue和postValue后一个在主线程执行，前者在当前线程执行。

```java
button.setOnClickListener(new OnClickListener() {
    @Override
    public void onClick(View v) {
        String anotherName = "John Doe";
        model.getCurrentName().setValue(anotherName);
    }
});
```

## 源码解析

先看一下LiveData相关的几个类：

### MutableLiveData类

```java
public class MutableLiveData<T> extends LiveData<T> {
    @Override
    public void postValue(T value) {
        super.postValue(value);
    }

    @Override
    public void setValue(T value) {
        super.setValue(value);
    }
}
```

继承于LiveData主要提供一层泛型，让LiveData支持任何数据类型，主要功能还是在LiveData中实现。

### Observer接口

```java
public interface Observer<T> {
    /**
     * Called when the data is changed.
     * @param t  The new data
     */
    void onChanged(@Nullable T t);
}
```

主要提供onChanged(T t)接口给LiveData通知数据变化，没什么好看的。主要看LiveData的observe()方法，怎么讲Observer对象绑定在一起。

### LiveData类

```java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
    // 如果生命周期组件的state已经destory，则不绑定
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    // 创建lifecycle于observer绑定对象，wrapper模式
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    // 判断observer是否已经添加到其他生命周期组件中，一个observer只能绑定一个
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    // 生命周期owner添加wrapper观察者
    owner.getLifecycle().addObserver(wrapper);
}
```

如果你不知道LifecycleOwner是啥玩意儿，可能看注释有点懵逼，你可以简单理解为Activity，通过LifecycleOwner可通过getLifecycle()获取到当前抓紧的Lifecyle对象，这个对象可以感知Activity等组件的生命周期变化，所以你要获取activity的生命周期，不一定要拿到activity实例，直接获取跟activity绑定的Lifecycle对象即可。扯远了，继续看LifecycleBoundObserver对象

创建LifecycleBoundObserver之后，最后会调用：

`owner.getLifecycle().addObserver(wrapper)`

这样生命周期组件的生命周期变化就能通知到Observer对象，至于为什么这里要加一层wrapper，当然是由一些通用的逻辑需要在wrapper中处理，而不是普通的Observer对象。

### LifecycleBoundObserver类

它是LiveData的内部类，并且继承于**ObserverWrapper**，实现**GenericLifecycleObserver接口**

```java
class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
    @NonNull final LifecycleOwner mOwner;

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<T> observer) {
        super(observer);
        mOwner = owner;
    }

    @Override
    boolean shouldBeActive() {
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }

    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
        activeStateChanged(shouldBeActive());
    }

    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }

    @Override
    void detachObserver() {
        mOwner.getLifecycle().removeObserver(this);
    }
}
```

### ObserverWrapper类

也是LiveData的内部类

```java
private abstract class ObserverWrapper {
    final Observer<T> mObserver;
    boolean mActive;
    int mLastVersion = START_VERSION;

    ObserverWrapper(Observer<T> observer) {
        mObserver = observer;
    }

    abstract boolean shouldBeActive();

    boolean isAttachedTo(LifecycleOwner owner) {
        return false;
    }

    void detachObserver() {
    }

    void activeStateChanged(boolean newActive) {
        if (newActive == mActive) {
            return;
        }
        // immediately set active state, so we'd never dispatch anything to inactive
        // owner
        mActive = newActive;
        boolean wasInactive = LiveData.this.mActiveCount == 0;
        LiveData.this.mActiveCount += mActive ? 1 : -1;
        if (wasInactive && mActive) {
            onActive();
        }
        if (LiveData.this.mActiveCount == 0 && !mActive) {
            onInactive();
        }
        if (mActive) {
            dispatchingValue(this);
        }
    }
}
```

### GenericLifecycleObserver接口

```
public interface GenericLifecycleObserver extends LifecycleObserver {
    /**
     * Called when a state transition event happens.
     *
     * @param source The source of the event
     * @param event The event
     */
    void onStateChanged(LifecycleOwner source, Lifecycle.Event event);
}
```

这是Lifecyle的组件，继承自LifecycleObserver，生命周期组件的所有生命周期变化都会通知onStateChanged()，LifecycleBoundObserver重写这个接口即可获取到Activity等组件的生命周期，并处理自己的逻辑。比如：

LifecycleBoundObserver类

```java
		@Override
        public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
            if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
                // 如果组件destory了，删除observer
                removeObserver(mObserver);
                return;
            }
            activeStateChanged(shouldBeActive());
        }
```

ObserverWrapper类

```java
void activeStateChanged(boolean newActive) {
    if (newActive == mActive) {
        return;
    }
    // immediately set active state, so we'd never dispatch anything to inactive
    // owner
    mActive = newActive;
    boolean wasInactive = LiveData.this.mActiveCount == 0;
    LiveData.this.mActiveCount += mActive ? 1 : -1;
    // active count从0变为1，通知active
    if (wasInactive && mActive) {
        // 通知active
        onActive();
    }
    // 所有active的observer数从1变为0，通知inactive
    if (LiveData.this.mActiveCount == 0 && !mActive) {
        // 通知inactive
        onInactive();
    }
    if (mActive) {
        // active的时候dispatch数据，这样当Activity在后台切前台的时候能立即获取到在后台时数据发生的变化
        dispatchingValue(this);
    }
}
```

shouldBeActive()时一个抽象方法，有两个实现：

LifecycleBoundObserver：从Lifecycle组件获取，至少需要**STARTED**以上才active

```java
@Override
boolean shouldBeActive() {
    return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
}
```

AlwaysactiveObserver: 总是active

```java
@Override
boolean shouldBeActive() {
    return true;
}
```

### 通知数据变化

我们知道 setValue和postValue两个方法LiveData用来通知数据变化，postValue只不过切换到主线程调用setValue而已。来看看setValue():

```java
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    // 数据版本升级
    mVersion++;
    // 保存数据，用一个成员变量，因为LiveData只管理一个抽象的数据
    mData = value;
    // 通知数据变化
    dispatchingValue(null);
}
```

```java
private void dispatchingValue(@Nullable ObserverWrapper initiator) {
    // 如果正在dispatch,设置变量dispatch无效为true，直接返回
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    // 开始dispatch
    mDispatchingValue = true;
    do {
        // 无效设置成false，表示有效
        mDispatchInvalidated = false;
        // 如果传入发起人，只通知传入的initiator
        if (initiator != null) {
            considerNotify(initiator);
            initiator = null;
        } else {
            // 遍历mObservers中所有的ObserverWrapper，通知所有LiveData的Observer
            for (Iterator<Map.Entry<Observer<T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ){				  // 通知变化
                considerNotify(iterator.next().getValue());
                // 如果在dispatch数据的时候再次发生dispatchingValue()，剩下的Observer直接跳过，因为下一轮的数据更新
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    // 如果在dispatch期间没有被新数据打断过，这里就直接过了，如果被打断过，则继续重新notify新的数据
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}
```

上面dispatch的逻辑有点绕，但是注释写得很清楚，就是在dispatch期间livedata又有新数据来了，打断并全部通知新数据的逻辑，我觉得这里还是写得很妙的，可以学习学习。

```java
private void considerNotify(ObserverWrapper observer) {
    // 如果observer状态没有active直接返回
    if (!observer.mActive) {
        return;
    }
    // 如果observer当前的状态不是active，直接返回（两种实现，前面说过）
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    // 如果当前observer的版本号比新数据的还新，直接return
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    // 设置observer中数据的版本号
    observer.mLastVersion = mVersion;
    // 调用observer的 onChanged回调
    observer.mObserver.onChanged((T) mData);
}
```

看到这里我们就明白了，生命周期不active的组件是怎么不通知数据变化的。这里每个observer中还有数据版本的概念，保证所有的订阅者都能拿到最新的数据。

### 总结

到这里，LiveData的主要逻辑我们就梳理完了，还有一些细节请读者就自己去阅读了。主要把握几个点：
1. LiveData通过LifecycleOwner获取到Lifecycle对象，并通过实现 GenericLifecycleObserver 监听onStateChanged获取到生命周期组件的生命周期变化，进而可以控制对active的组件才通知数据变化。
2. LiveData对数据版本，以及短时间多次数据变化都有做控制，active的数据Observer都获取到最新的数据。
3. 这里有两个Observer，一个是数据Observer是Activity中注册监听LiveData数据变化的；宁外一个是LiveData为了获取生组件生命周期实现的Observer。不要搞混了。
4. 当生命周期组件从后台切前台等操作，当它active的时候会自动收到最新版本的LiveData数据变化，也是通过onStateChanged监听实现的。

然后按照惯例，画一张图：
![](./image-20190324221458032.png)