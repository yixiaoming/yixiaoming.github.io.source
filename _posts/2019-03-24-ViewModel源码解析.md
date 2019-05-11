---
title: ViewModel源码解析
tags:
  - Android
categories:
  - Android源码解析系列
comments: true
date: 2019-03-24 09:31:41
---

## 介绍

本片文章主要解读Android官方推荐组件ViewModel的底层实现原理，ViewModel作为Activity/Fragment等组件的数据容器，可以避免我们在View层自己创建并保存数据，都交给ViewModel管理，在横竖屏切换等场景也能很好适配。让我们再不需要再onSaveInstanceState()来保存临时变量，并且ViewModel可能很好的适应组件的生命周期，在Activity等组件onDestroy()时自动清除数据避免内存泄漏。

先来一张简图：
![image-20190324122440712](/images/ViewModel源码解析/image-20190324122440712.png)

<!-- more -->

## 源码解读

### 使用方法（官网例子）

第1步：声明ViewModel组件

```java
public class MyViewModel extends ViewModel {
    private MutableLiveData<List<User>> users;
    public LiveData<List<User>> getUsers() {
        if (users == null) {
            users = new MutableLiveData<List<User>>();
            loadUsers();
        }
        return users;
    }

    private void loadUsers() {
        // Do an asynchronous operation to fetch users.
    }
}
```

第2步：在Activity中获取ViewModel组件，并获取ViewModel中存储的内容

```java
public class MyActivity extends AppCompatActivity {
    public void onCreate(Bundle savedInstanceState) {
        // Create a ViewModel the first time the system calls an activity's onCreate() method.
        // Re-created activities receive the same MyViewModel instance created by the first activity.

        MyViewModel model = ViewModelProviders.of(this).get(MyViewModel.class);
        model.getUsers().observe(this, users -> {
            // update UI
        });
    }
}
```

### 源码分析

首先需要注意：

1. ViewModel这家伙不是我们直接new出来的，而是通过ViewModelProviders创建来的，所以其不受Activity等组件生命周期的限制。
2. 如果对LiveData不了解，可以直接忽略。

####  ViewModelProviders 的创建

那么ViewModel是怎么通过ViewModelProviders创建的呢？首先看`ViewModelProviders.of(this)`，

```java
public static ViewModelProvider of(@NonNull FragmentActivity activity,
        @Nullable Factory factory) {
    Application application = checkApplication(activity);
    if (factory == null) {
        factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
    }
    return new ViewModelProvider(ViewModelStores.of(activity), factory);
}
```

创建了一个**ViewModelProvider**对象，有两个参数：

1. **ViewModelStores.of(activity)** 最后生成的是一个 **ViewModelStore** 对象
2.  **ViewModelProvider.AndroidViewModelFactory.getInstance(application)**

先看ViewModelStore的创建：

```java
public class ViewModelStores {

    private ViewModelStores() {
    }

    /**
     * Returns the {@link ViewModelStore} of the given activity.
     *
     * @param activity an activity whose {@code ViewModelStore} is requested
     * @return a {@code ViewModelStore}
     */
    @NonNull
    @MainThread
    public static ViewModelStore of(@NonNull FragmentActivity activity) {
        if (activity instanceof ViewModelStoreOwner) {
            return ((ViewModelStoreOwner) activity).getViewModelStore();
        }
        return holderFragmentFor(activity).getViewModelStore();
    }

    /**
     * Returns the {@link ViewModelStore} of the given fragment.
     *
     * @param fragment a fragment whose {@code ViewModelStore} is requested
     * @return a {@code ViewModelStore}
     */
    @NonNull
    @MainThread
    public static ViewModelStore of(@NonNull Fragment fragment) {
        if (fragment instanceof ViewModelStoreOwner) {
            return ((ViewModelStoreOwner) fragment).getViewModelStore();
        }
        return holderFragmentFor(fragment).getViewModelStore();
    }
}
```

通过of方法，

1. 如果传入的Activity或Fragment实现了ViewModelStoreOwner接口，则直接返回getViewModelStore()返回的ViewModelStore
2. 如果传入的组件不是ViewModelStoreOwner，需要获取HolderFragment的ViewModelStore。这是一种兼容方案。这里看一下Activity的方案，同理还有Fragment的，大同小异。

```java
HolderFragment holderFragmentFor(FragmentActivity activity) {
    FragmentManager fm = activity.getSupportFragmentManager();
    // 通过tag从FragmentManager中找Fragment
    HolderFragment holder = findHolderFragment(fm);
    if (holder != null) {
        return holder;
    }
    // 从创建了但是还没有commit的fragment缓存中读取，真正清除实在HolderFragment的onCreate执行后，即承载HolderFragment的容器onCreate()之后
    holder = mNotCommittedActivityHolders.get(activity);
    if (holder != null) {
        return holder;
    }
	// 注册生命周期回调，在onDestroy的时候清除 mNotCommittedActivityHolders中的对象。
    if (!mActivityCallbacksIsAdded) {
        mActivityCallbacksIsAdded = true;
        activity.getApplication().registerActivityLifecycleCallbacks(mActivityCallbacks);
    }
    // 创建HolderFragment并add到FragmentManager中
    holder = createHolderFragment(fm);
    mNotCommittedActivityHolders.put(activity, holder);
    return holder;
}
```

到这里我们获取到了ViewModelOwner，然后通过getViewModelStore()获取到了ViewModelStore对象。ViewModelStoreOwner就是一个提供ViewModelStore的接口：

```java
public interface ViewModelStoreOwner {
    /**
     * Returns owned {@link ViewModelStore}
     *
     * @return a {@code ViewModelStore}
     */
    @NonNull
    ViewModelStore getViewModelStore();
}
```

我们来看FragmentActivity对这个接口的实现：

```java
FragmentActivity extends SupportActivity implements ViewModelStoreOwner,...{
    @NonNull
  public ViewModelStore getViewModelStore() {
    if (this.getApplication() == null) {
      throw new IllegalStateException("Your activity is not yet attached to the Application instance. You can't request ViewModel before onCreate call.");
    } else {
      if (this.mViewModelStore == null) {
        // 这里很精髓，会将ViewModel绑定到不因为configuration变化导致Activity销毁被清空的NonConfigurationInstances下
        FragmentActivity.NonConfigurationInstances nc = (FragmentActivity.NonConfigurationInstances)this.getLastNonConfigurationInstance();
        if (nc != null) {
          this.mViewModelStore = nc.viewModelStore;
        }

        if (this.mViewModelStore == null) {
          this.mViewModelStore = new ViewModelStore();
        }
      }

      return this.mViewModelStore;
    }
  }
}
```

至此终于获取到ViewModelStore对象，是通过ViewModelStoreOwner接口来获取，需要Activity或者Fragment直接实现接口，如果不是先，系统会为我们的组件添加HolderFragment来做兼容。至于ViewModelStore有什么作用，到后面我们再看。现在看第二个参数 **AndroidViewModelFactory**的创建。这个factory类其实是ViewModelProvider的静态内部类，而且是个单例：

```java
public static class AndroidViewModelFactory extends ViewModelProvider.NewInstanceFactory {

    private static AndroidViewModelFactory sInstance;

    /**
     * Retrieve a singleton instance of AndroidViewModelFactory.
     *
     * @param application an application to pass in {@link AndroidViewModel}
     * @return A valid {@link AndroidViewModelFactory}
     */
    @NonNull
    public static AndroidViewModelFactory getInstance(@NonNull Application application) {
        if (sInstance == null) {
            sInstance = new AndroidViewModelFactory(application);
        }
        return sInstance;
    }

    private Application mApplication;

    /**
     * Creates a {@code AndroidViewModelFactory}
     *
     * @param application an application to pass in {@link AndroidViewModel}
     */
    public AndroidViewModelFactory(@NonNull Application application) {
        mApplication = application;
    }

    @NonNull
    @Override
    public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
        if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
            //noinspection TryWithIdenticalCatches
            try {
                return modelClass.getConstructor(Application.class).newInstance(mApplication);
            } catch (NoSuchMethodException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (IllegalAccessException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (InstantiationException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (InvocationTargetException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            }
        }
        return super.create(modelClass);
    }
}
```

唯一一个重要方法就是 create方法，通过反射直接创建ViewModel对象。

#### ViewModelProvider的功能

刚刚我们分析了通过`ViewModelProviders.of(this)`创建了一个ViewModelProvider对象，然后我们再看Activity中的代码 `model = ViewModelProviders.of(this).get(NameViewModel.class);`下一步就是通过 ViewModelProvider的get(Class)方法来获取ViewModel对象：

ViewModelProvider类：

```java
@NonNull
@MainThread
public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
    // 创建ViewModel中内存中的key
    return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}

@NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        // 从ViewModelStore中通过key获取ViewModel对象
        ViewModel viewModel = mViewModelStore.get(key);
		// 通过key获取的对象与想要的Class相同直接返回
        if (modelClass.isInstance(viewModel)) {
            return (T) viewModel;
        } else {
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }
		// 通过ViewModelFactory创建ViewModel类，并通过key-value添加到store中
        viewModel = mFactory.create(modelClass);
        mViewModelStore.put(key, viewModel);
        //noinspection unchecked
        return (T) viewModel;
    }
```

这里我们在上一步分析的两个关键类都起作用了:

1. ViewModelStore:通过**DEFAULT-KEY+类名**存储ViewModel对象。

2. 通过ViewModelFactory创建ViewModel对象。

**ViewModelStore类：**

```java
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    /**
     *  Clears internal storage and notifies ViewModels that they are no longer used.
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.onCleared();
        }
        mMap.clear();
    }
}
```

其就是一个HashMap的ViewModel容器，并且提供clear方法清空所有内容，既然要清空，那么肯定就在一个地方onDestroy()中做，所以看看FragmentActivity的onDestroy()，果然不出所料：

```
protected void onDestroy() {
    super.onDestroy();
    if (this.mViewModelStore != null && !this.isChangingConfigurations()) {
      this.mViewModelStore.clear();
    }

    this.mFragments.dispatchDestroy();
  }
```

这里还对要求 **!this.isChangingConfigurations()** ，就是如果因为系统configuration导致onDestroy这种情况不清楚ViewModel，这就是为什么横竖屏切换等场景Activity重建了但是ViewModel依然健在。

#### 再捋一捋逻辑

再看看这句代码：`model = ViewModelProviders.of(this).get(NameViewModel.class);`

1. 首先第一步**ViewModelProviders.of(this)**获取ViewModelProvider对象，这个provider有两部分组成 ViewModelStore 和 AndroidViewModelFactory

2. 获取ViewModelStore是通过ViewModelStoreOwner接口来的，每个组件自己去实现并创建自己的ViewModelStore，FragmentActivity，support.v4.app.Framgnet天然支持。如果传入的组件不支持，ViewModelStores 会创建一个隐藏的HolderFragment为你达到目的。
3. 通过**ViewModelProvider.get(xxx.class)**会先到每个组件的ViewModelStore中去找，如果找不到通过AndroidViewModelFactory创建并放到ViewModelStore中。
4. 当生命周期组件onDestroy的时候会清理ViewModelStore中的内容防止内存泄漏了，但是由于ConfigurationChange导致的onDestroy不会清理，因为会将ViewModel绑定到不因为configuration变化导致Activity销毁被清空的NonConfigurationInstances下，下一次重建依然会被获取到。

### ViewModel的用法

官网就给出了3大ViewModel的用途：

1. 结合LiveData，为View组件提供数据，并保证不因为Configuration变化导致数据丢失。
2. Fragment之间数据共享。`SharedViewModel model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);`同一个Activity下发不同Fragment可以通过传入Activity的context获取到相同的ViewModel对象。**Why？同一个Activity获取同一个ViewModelStore，相同的.class获取到的就是相同的ViewModel。**
3. 替换Loaders，至于这玩意儿是啥，我没用过。看图大概的意思就是：Loader从DataSource获取数据，通知给LoaderManager，LoaderManager通知UI Controller更新。使用ViewModel+LiveData天然就可以干这个事情。

