---
layout: post
title:  "Jetpack系列之ViewModel"
date:   2018-09-03 15:32:54
categories: Android
tags: ViewModel 
---

* content
{:toc}

本文记录[Android Jetpack][1]中`Architecture`部分的`ViewModel`以及工作原理。




# Android Jetpack系列之ViewModel

[Android Jetpack][3]是一系列助力更容易打造卓越 Android 应用的工具和组件。这些组件能帮助您遵循最佳实践、免除编写繁复的样板代码并简化复杂任务，从而使您可以专注于最核心的代码逻辑。

`Jetpack` 中的架构指南由 `Android` 开发中四个关键领域中的一系列代码库和工具提供支持。它们分别是**基础（Foundation）、架构（Architecture）、行为（Behavior）和 UI**。每个 `Jetpack` 组件均可单独采用，但它们依然可以流畅地协作。其中 `androidx.*` 库与 `Framework API`解耦，这能够提供向后兼容的同时，也能更频繁地更新。

本文记录[Android Jetpack][4]架构组件（Architecture）中的[ViewModel][5]。

## ViewModel概述
`ViewModel`用来管理UI交互的数据，它保存的数据不会因为`Activity`配置改变而重新生成，比如屏幕旋转会重新触发`onCreate()`，但是`ViewModel`保存的数据不会丢失。

`Activity`和`Fragment`的生命周期是由`Framework`管理，`Framework`会在不在用户控制的情况下对`Activity`、`Fragment`进行销毁和重建。
如果`Activity`、`Fragment`销毁或者重建之后，存储在当中的数据将会丢失，简单的数据可以通过`onSaveInstanceState()`跟`onCreate()`来恢复，但是大数据，比如位图，或者一个列表却不适用，现在这些数据可以保存到`ViewModel`中。
![效果图][2]

## 实现ViewModel
实现`ViewModel`只需要继承`ViewModel`即可，如：
```kotlin
class TextViewModel : ViewModel() {
    var text: String? = null
        get() {
            if (field == null) {
                field = asyncText()
            }
            return field
        }

    private fun asyncText(): String = "This is a Text"
}
```
然后再通过`ViewModelProviders`来获取：
```kotlin
class ViewModelActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_view_model)
        val textViewModel = ViewModelProviders.of(this)
                .get(TextViewModel::class.java)

        val modelTv = findViewById<TextView>(R.id.tv_model)
        modelTv.text = textViewModel.text
    }
}
```
如果`Activity`重新创建，获取到的`textViewModel`是同一个实例，当`Activity` finished之后会调用`ViewModel`的`onCleared`，用于回收资源,下文追踪源码分析。

**注意：`ViewModel`不能有对`View`的引用，或者其他对`Activity Context`有引用的类。**

如果需要用到`Application Context`可以选择继承`AndroidViewModel`。
```kotlin
public class AndroidViewModel extends ViewModel {
    @SuppressLint("StaticFieldLeak")
    private Application mApplication;

    public AndroidViewModel(@NonNull Application application) {
        mApplication = application;
    }

    /**
     * Return the application.
     */
    @SuppressWarnings("TypeParameterUnusedInFormals")
    @NonNull
    public <T extends Application> T getApplication() {
        //noinspection unchecked
        return (T) mApplication;
    }
}
```

## ViewModel的生命周期

![ViewModel的生命周期][6]

通常情况下会在`Activity`的`onCreate()`方法获取`ViewModel`，此后无论`onCreate()`调用多少次，获取到的`ViewModel`都是同一个实例。

## ViewModel源码分析
```kotlin
public abstract class ViewModel {
    /**
     * This method will be called when this ViewModel is no longer used and will be destroyed.
     * <p>
     * It is useful when ViewModel observes some data and you need to clear this subscription to
     * prevent a leak of this ViewModel.
     */
    @SuppressWarnings("WeakerAccess")
    protected void onCleared() {
    }
}
```
`ViewModel`是一个抽象类，中间只有一个可选择实现的方法`onCleared()`，该方法会在重复缓存当前`ViewModel`到`ViewModelStore`中或者`HolderFragment`的`onDestory()`被调用，用于回收资源。

## ViewModelProviders.of(xxx).get(xxx)源码分析
对于上文中出现的代码：
```kotlin
val textViewModel = ViewModelProviders.of(this).get(TextViewModel::class.java)
```
`ViewModelProviders.of(this)`的操作：
一共有4个重载方法
```java
@NonNull
@MainThread
public static ViewModelProvider of(@NonNull Fragment fragment) {
        ViewModelProvider.AndroidViewModelFactory factory =
                ViewModelProvider.AndroidViewModelFactory.getInstance(
                        checkApplication(checkActivity(fragment)));
        return new ViewModelProvider(ViewModelStores.of(fragment), factory);
    }
    
@NonNull
@MainThread
public static ViewModelProvider of(@NonNull FragmentActivity activity) {
        ViewModelProvider.AndroidViewModelFactory factory =
                ViewModelProvider.AndroidViewModelFactory.getInstance(
                        checkApplication(activity));
        return new ViewModelProvider(ViewModelStores.of(activity), factory);
    }
    
@NonNull
@MainThread
public static ViewModelProvider of(@NonNull Fragment fragment, @NonNull Factory factory) {
        checkApplication(checkActivity(fragment));
        return new ViewModelProvider(ViewModelStores.of(fragment), factory);
    }
    
@NonNull
@MainThread
public static ViewModelProvider of(@NonNull FragmentActivity activity,
            @NonNull Factory factory) {
        checkApplication(activity);
        return new ViewModelProvider(ViewModelStores.of(activity), factory);
    }
```

该方法必须在主线程中调用，前两个方法不需要提供`Factory`，会实现默认的factory。
```java
ViewModelProvider.AndroidViewModelFactory factory = ViewModelProvider.AndroidViewModelFactory.getInstance(checkApplication(activity));
```
该方法会通过`AndroidViewModelFactory#getInstance()`返回一个`AndroidViewModelFactory`的单例，何为`AndroidViewModelFactory`？
```java
 public static class AndroidViewModelFactory extends ViewModelProvider.NewInstanceFactory {

        private static AndroidViewModelFactory sInstance;

        /**
         * Retrieve a singleton instance of AndroidViewModelFactory.
         *
         * @param application an application to pass in {@link AndroidViewModel}
         * @return A valid {@link AndroidViewModelFactory}
         */
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
`AndroidViewModelFactory`继承自`NewInstanceFactory`。
```java
 public static class NewInstanceFactory implements Factory {

        @SuppressWarnings("ClassNewInstance")
        @NonNull
        @Override
        public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
            //noinspection TryWithIdenticalCatches
            try {
                return modelClass.newInstance();
            } catch (InstantiationException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (IllegalAccessException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            }
        }
    }
```
该类实现了`Factory`接口，该接口只有一个方法`create()`用于创建新的实例。
回到方法
```java
ViewModelProvider.AndroidViewModelFactory factory = ViewModelProvider.AndroidViewModelFactory.getInstance(checkApplication(activity));
```
该方法通过`AndroidViewModelFactory.getInstance(checkApplication(activity))`返回`AndroidViewModelFactory`的单例。其中`checkApplication(activity)`用于返回`Activity/Fragment`绑定的`Application`，如果没有绑定`Application`则会抛出异常。

再回到最初的`of`方法。
```java
@MainThread
public static ViewModelProvider of(@NonNull FragmentActivity activity) {
        ViewModelProvider.AndroidViewModelFactory factory = ViewModelProvider.AndroidViewModelFactory.getInstance(checkApplication(activity));
        return new ViewModelProvider(ViewModelStores.of(activity), factory);
    }
```
```java
return new ViewModelProvider(ViewModelStores.of(activity), factory);
```
该句代码中的`ViewModelStores.of(activity)`会返回一个`ViewModelStore`，**该类用于存储`ViewModel`**。
`ViewModelStores.of(activity)`有两个重载方法。
```java
@NonNull
@MainThread
public static ViewModelStore of(@NonNull FragmentActivity activity) {
        if (activity instanceof ViewModelStoreOwner) {
            return ((ViewModelStoreOwner) activity).getViewModelStore();
        }
        return holderFragmentFor(activity).getViewModelStore();
    }


@NonNull
@MainThread
public static ViewModelStore of(@NonNull Fragment fragment) {
        if (fragment instanceof ViewModelStoreOwner) {
            return ((ViewModelStoreOwner) fragment).getViewModelStore();
        }
        return holderFragmentFor(fragment).getViewModelStore();
    }
```
这两个构造函数除了入参不一样，其他基本是一样的，先判断`Activity/Fragment`是否实现了`ViewModelStoreOwner`接口，实现该接口要求实现`getViewModelStore()`方法并且返回一个`ViewStore`。如果没有实现该接口则会调用`holderFragmentFor(activity/fragment).getViewModelStore()`返回`ViewModelStore()`。

`holderFragmentFor(activity/fragment).getViewModelStore()`方法位于`HolderFragment`中。
```java
/**
     * @hide
     */
    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    public static HolderFragment holderFragmentFor(FragmentActivity activity) {
        return sHolderFragmentManager.holderFragmentFor(activity);
    }

    /**
     * @hide
     */
    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    public static HolderFragment holderFragmentFor(Fragment fragment) {
        return sHolderFragmentManager.holderFragmentFor(fragment);
    }
```
两个方法会返回`HolderFragment`，`sHolderFragmentManager`是`HolderFragmentManager`用于管理`HolderFragment`，其中的`holderFragmentFor(Activity)`方法如下：
```java
HolderFragment holderFragmentFor(FragmentActivity activity) {
            FragmentManager fm = activity.getSupportFragmentManager();
            //通过HOLDER_TAG获取已经添加过的HolderFragment
            HolderFragment holder = findHolderFragment(fm);
            if (holder != null) {
                return holder;
            }
            //mNotCommittedActivityHolders是一个Map，用于存储HolderFragment，key为Activity
            holder = mNotCommittedActivityHolders.get(activity);
            if (holder != null) {
                return holder;
            }
            //用于判断是否注册ActivityLifeCycle回调，没有则注册，用于Activity销毁的时候移除对应的Activity以及触发对应ViewModel的onCleared()
            if (!mActivityCallbacksIsAdded) {
                mActivityCallbacksIsAdded = true;
                activity.getApplication().registerActivityLifecycleCallbacks(mActivityCallbacks);
            }
            //创建新的HolderFragment并且为之设置TAG为HOLDER_TAG
            holder = createHolderFragment(fm);
            //将创建的HolderFragment放进mNotCommittedActivityHolders用于下次直接get()
            mNotCommittedActivityHolders.put(activity, holder);
            return holder;
        }
```
再看`holderFragmentFor(fragment).getViewModelStore()`就很清晰了：
```java
    private ViewModelStore mViewModelStore = new ViewModelStore();
    
    public HolderFragment() {
        //当前Fragment绑定的Activity被recreate时，Fragment的onDestroy()和onCreate()不会调用
        setRetainInstance(true);
    }
    @Override
    public void onDestroy() {
        super.onDestroy();
        mViewModelStore.clear();
    }

    @NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        return mViewModelStore;
    }

```
其实就是直接`new ViewModelStore()`了，何为`ViewModelStore`？
```java
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.get(key);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
        mMap.put(key, viewModel);
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
这个就很简单了。
获取到了`ViewModelStore`，再回到最初的`ViewModelProviders.of()`方法。
```java
public static ViewModelProvider of(@NonNull FragmentActivity activity) {
        ViewModelProvider.AndroidViewModelFactory factory =
                ViewModelProvider.AndroidViewModelFactory.getInstance(
                        checkApplication(activity));
        return new ViewModelProvider(ViewModelStores.of(activity), factory);
    }
```
下面这句返回一个`ViewModelProvider`，该类没有父类，追踪源码可知。
```java
 public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
        mFactory = factory;
        this.mViewModelStore = store;
    }
```
只是设置了`mFactory`跟`ViewModelStore`，上文分析可知这两个参数的由来。

再回到获取`ViewModel`方法中的`ViewModelProviders.of(this).get(TextViewModel::class.java)`
分析`get()`，上文可知`of`返回的是`ViewModelProvider`，所以直接打开`ViewModelProvider#get()`方法。
```java
    @NonNull
    public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }
```
其中`canonicalName`作为类的唯一标识。DEFAULT_KEY为`android.arch.lifecycle.ViewModelProvider.DefaultKey`。点开重载方法`get(DEFAULT_KEY + ":" + canonicalName, modelClass);`
```java
@NonNull
@MainThread
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        //通过ViewModelStore来获取已存在的ViewModel
        ViewModel viewModel = mViewModelStore.get(key);
        //如果modelClass是上面viewModel的实例则直接返回
        if (modelClass.isInstance(viewModel)) {
            //noinspection unchecked
            return (T) viewModel;
        } else {
            //noinspection StatementWithEmptyBody
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }
        //通过Factory创建一个ViewModel，前文中的AndroidModelFacotry中的create()方法
        viewModel = mFactory.create(modelClass);
        //将当前ViewModel存储到ViewModelStore中
        mViewModelStore.put(key, viewModel);
        //noinspection unchecked
        return (T) viewModel;
    }
```
至此，`ViewModelProviders.of(xx).get(xxx)`分析完毕。


  [1]: https://www.youtube.com/watch?v=LmkKFCfmnhQ&t=42s
  [2]: https://github.com/qfxl/qfxl.github.io/blob/master/gif/viewmodel.gif
  [3]: https://www.youtube.com/watch?v=LmkKFCfmnhQ&t=42s
  [4]: https://www.youtube.com/watch?v=LmkKFCfmnhQ&t=42s
  [5]: https://developer.android.google.cn/topic/libraries/architecture/viewmodel
  [6]: https://developer.android.google.cn/images/topic/libraries/architecture/viewmodel-lifecycle.png