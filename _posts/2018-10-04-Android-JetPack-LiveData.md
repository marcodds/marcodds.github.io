---
layout: post
title:  "Jetpack系列之LiveData"
date:   2018-09-13 15:32:54
categories: Android
tags: LiveData 
---

* content
{:toc}

本文记录[Android Jetpack][1]中`Architecture`部分的[LiveData][2]以及工作原理。




# Android Jetpack系列之LiveData

## LiveData概述
[LiveData][3]是一个可观察数据的持有类，与常规的`Observable`不同，`LiveData`能够感知生命周期变化，他能与`Activity`或者`Fragment`、`Service`的生命周期关联，这能确保`LiveData`只在`Activity`或者`Fragment`生命周期处于活跃的时候刷新数据。

`LiveData`将一个由`Observer`类表示的观察者视为生命周期处于`STARTED`或者`RESUMED`状态。`LiveData`只会通知处于活跃状态的`Observer`更新数据，非活跃的`Observer`不会接收到通知。
 
注册`Observer`观察`LiveData`需要一个`LifecyleOwner`，`LifecyleOwner`是一个接口，这个`LifecycleOwner`状态变更为`DESTROYED`时候会移除与`LifecycleOwner`对应的`Observer`，通常情况下`LifecycleOwner`是一个`Fragment`或者`Activity`，因为在新的`SupportActivity`、`Fragment`都实现了`LifeCycleOwner`接口。有了这种对应关系就不用再担心操作发生内存泄漏了。
```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        liveData.observe(this, Observer {
            
        })
    }
}

```
## 使用LiveData的优点

* 确保UI更新符合数据状态
    `LiveData`遵循观察者模式，`Observer`生命周期状态更改时`LiveData`会通知对应的`Observer`，每次`LiveData`数据更改时，对应的观察者可以在每次更新时更新UI。

* 没有内存泄漏
`Observer`绑定的`LifecycleOwner`对象在`LifecycleOwner`销毁时自动解除绑定。

* 不会再有由于`Activity` `Stop`时引起的crash
如果`Observer`的生命周期处于非活跃状态，例如`Activity`退到后台，那`Observer`不会再接收到`LiveData`的数据更新。

* 不需要手动处理生命周期
UI组件只观察与其有关的数据，不会自动停止、恢复观察对应数据，`LiveData`自动管理这些，因为它会跟这些生命周期联动。

* 始终保持最新数据
如果生命周期变为非活动状态，则会在再次变为活动状态时接收最新数据。例如，后台活动在返回前台后立即收到最新数据。

* 更好的处理配置更改
如果`Activity`或者`Fragment`由于配置发生改变而重新`OnCreate`，如屏幕旋转，`Observer`接收到的始终是最新数据。

* 资源共享
我们可以使用单例模式来扩展`LiveData`然后包装系统服务，然后在应用内这个`LiveData`是共享的。

## 使用LiveData

### 创建LiveData对象
`LiveData`是一个包装，真正需要的是其包装的数据，通常情况下`LiveData`与`ViewModel`一起使用。

```kotlin
class NameViewModel : ViewModel() {
    var names: MutableLiveData<List<String>>? = null
    private set
    get() {
        if (field == null) {
            field = MutableLiveData()
        }
        return field
    }
}
```

### 观察LiveData对象
在大多数情况，在`onCreate`中观察`LiveData`是最好的，因为避免了`onResume`的重复调用，同时确保`Activity`或者`Fragment`能在处于活跃状态时立即显示数据，只要应用组件处理`STARTED`状态，它就会从`LiveData`观察的对象中获取最近值。
此外，`LiveData`只有在数据更改，并且只有处于活跃的`Observer`才能接收到更新。不过有一个例外，当`Observer`从非活跃状态变更改活跃状态时也会接受到更新。
```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        val mNameViewModel = ViewModelProviders.of(this)
                .get(NameViewModel::class.java)

        val namesTv = findViewById<TextView>(R.id.tv_names)
        mNameViewModel.names?.observe(this, Observer {
            namesTv.text = it.toString()
        })
    }
}
```

### 更新LiveData对象
`LiveData`没有公开的方法更新持有的数据，只有[MutableLiveData][4]提供了`setValue(T)`和`postValue(T)`用于更新持有的数据，`setValue(T)`必须在主线程中调用，而`postValue(T)`可以在子线程中调用，通常`MutableLiveData`用于`ViewModel`然后将`ViewModel`唯一的不可变`LiveData`对象暴露给观察者。

更新`LiveData`中的对象的值。
```kotlin
 findViewById<Button>(R.id.btn_update).apply {
            setOnClickListener {
                mNameViewModel.names?.value = ArrayList<String>().apply {
                    add("Anna")
                    add("Bruce")
                    add("qfxl")
                }
            }
        }
```
调用`setValue(T)`会回调观察者的`onChanged(T)`方法。

### 扩展LiveData
`LiveData`可以看做一个观察者，如果观察者的状态为`STARTED`或者`RESUMED`的时候可以认为`LiveData`正处于活跃的状态。
下面是一个模拟股票刷新的实例：
```kotlin
class StockLiveData : LiveData<BigDecimal>() {

    private object Holder {
        val INSTANCE = StockLiveData()
    }

    companion object {
        fun get(): StockLiveData {
            return Holder.INSTANCE
        }
    }

    private val dispatcherHandler = Handler(Looper.getMainLooper())

    private val updateRunnable = Runnable {
        value = BigDecimal.valueOf(Random().nextInt(2500).toLong())
        updateStockData()
    }

    override fun onActive() {
        updateStockData()
    }

    override fun onInactive() {
        dispatcherHandler.removeCallbacks(updateRunnable)
    }

    private fun updateStockData() {
        dispatcherHandler.postDelayed(updateRunnable, 1000)
    }
}
```
使用：
```kotlin
class StockActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_stock)
        val stockTv = findViewById<TextView>(R.id.tv_stock)
        val liveData = StockLiveData.get()
        liveData.observe(this, Observer {
            stockTv.text = "上证指数 $it"
        })
    }
}
```
在`Activity`不可见的时候自动解除绑定关系，在重新可见的时候会重新绑定。
//TODO 效果图

在上述示例中实现了几个比较重要的方法：

* `onActive`方法当`LiveData`有观察者的时候，可以在这个方法进行数据更新操作。
* `onInActive`方法当`LiveData`观察者从N-0的时候调用，就是当`LiveData`没有观察者的时候会被调用。
* `setValue(T)`用于更新`LiveData`持有的对象数据。

### 转换LiveData
有时候我们需要将`LiveData`持有的对象进行转换，比如`LiveData`原先持有的是一个`User`但是我想要的只有其中的`name`属性，这样就可以用到`Lifecycle`中提供的[Transformations][5]。

`Transformations.map()`

```kotlin
val userLiveData = UserLiveData()
val nameLiveData = Transformations.map(userLiveData) {
    it.name
}.observe(this, Observer {
    stockTv.text = it
})
```
`Transformations.switchMap()`
类似于`map()`但是需要返回一个`LiveData`
```kotlin
private fun getUser(id: String): LiveData<User> {
  ...
}
val userId: LiveData<String> = ...
val user = Transformations.switchMap(userId) { id -> getUser(id) }
```
以上为`LiveData`的基本用法。

## LiveData源码解析
从`LiveData`的`Observe`方法入手，这是`LiveData`跟`Observer`的关联入口。
```java
  @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        owner.getLifecycle().addObserver(wrapper);
    }
```
`LiveData#observe()`第一句代码。
```java
 if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
}
```
`owner`为`LifecycleOwner`，`owner.getLifecycle()`获取到的是`Lifecycle`，`LifecycleOwner`是一个接口。
```java
public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     *
     * @return The lifecycle of the provider.
     */
    @NonNull
    Lifecycle getLifecycle();
}
```

`Lifecycle`是一个抽象类，它只有一个已知子类`LifecycleRegistry`。

`getLifecycle().getCurrentState()`返回的是一个枚举值，它定义在`Lifecycle`中，每个枚举的意思都有详细的英文注释文档。

```java
  public enum State {
        /**
         * Destroyed state for a LifecycleOwner. After this event, this Lifecycle will not dispatch
         * any more events. For instance, for an {@link android.app.Activity}, this state is reached
         * <b>right before</b> Activity's {@link android.app.Activity#onDestroy() onDestroy} call.
         */
        DESTROYED,

        /**
         * Initialized state for a LifecycleOwner. For an {@link android.app.Activity}, this is
         * the state when it is constructed but has not received
         * {@link android.app.Activity#onCreate(android.os.Bundle) onCreate} yet.
         */
        INITIALIZED,

        /**
         * Created state for a LifecycleOwner. For an {@link android.app.Activity}, this state
         * is reached in two cases:
         * <ul>
         *     <li>after {@link android.app.Activity#onCreate(android.os.Bundle) onCreate} call;
         *     <li><b>right before</b> {@link android.app.Activity#onStop() onStop} call.
         * </ul>
         */
        CREATED,

        /**
         * Started state for a LifecycleOwner. For an {@link android.app.Activity}, this state
         * is reached in two cases:
         * <ul>
         *     <li>after {@link android.app.Activity#onStart() onStart} call;
         *     <li><b>right before</b> {@link android.app.Activity#onPause() onPause} call.
         * </ul>
         */
        STARTED,

        /**
         * Resumed state for a LifecycleOwner. For an {@link android.app.Activity}, this state
         * is reached after {@link android.app.Activity#onResume() onResume} is called.
         */
        RESUMED;

        /**
         * Compares if this State is greater or equal to the given {@code state}.
         *
         * @param state State to compare with
         * @return true if this State is greater or equal to the given {@code state}
         */
        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
```
`owner.getLifecycle().getCurrentState()`这句代码需要到`LifecycleOwner`实现类中去找对应方法，`LifecycleOwner`这个接口在系统中的`SupportActivity`、`Fragment`都有实现，这就是为什么在`Activity/Fragment`中调用`LiveData#Observe()`方法中可以传入`this`。
以`SupportActivity`为例，它是`Activity`的直接子类，并且实现了`LifecycleOwner`。
```java
public class SupportActivity extends Activity implements LifecycleOwner {
    xxx
    private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
    xxx
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
    xxx
}
```
`owner.getLifecycle().getCurrentState()`最终调用的地方为`LifecycleRegistry#getCurrentState()`，

```java
public class LifecycleRegistry extends Lifecycle {
    xxx
    private final WeakReference<LifecycleOwner> mLifecycleOwner;
      /**
     * Current state
     */
    private State mState;
    xx
    public LifecycleRegistry(@NonNull LifecycleOwner provider) {
        mLifecycleOwner = new WeakReference<>(provider);
        mState = INITIALIZED;
    }
    
    @Override
    public State getCurrentState() {
        return mState;
    }
}
```
第一次调用`getCurrentState`为`INITIALIZED`。

`LiveData#observe()`第二句代码。
```java
LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
```
`LifecycleBoundObserver`是`ObserverWrapper`的子类，何为`ObserverWrapper`？顾名思义是`Observer`的包装类。
```java
 private abstract class ObserverWrapper {
        final Observer<T> mObserver;
        boolean mActive;
        int mLastVersion = START_VERSION;

        ObserverWrapper(Observer<T> observer) {
            mObserver = observer;
        }
        /**
         * 需要实现的方法，是否是活跃状态。
         */
        abstract boolean shouldBeActive();

        boolean isAttachedTo(LifecycleOwner owner) {
            return false;
        }

        void detachObserver() {
        }
        /**
         * 状态更改
         */
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
点开`LifecycleBoundObserver`。
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
`LiveData#observe()`第三句代码。
```java
  // private SafeIterableMap<Observer<T>, ObserverWrapper> mObservers =
  //          new SafeIterableMap<>();
  //如果存在则直接返回，否则put到mObservers中。
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    //如果当前的ObserverWrapper不为空，并且当前传入的LifecycleOwner与ObserverWrapper原先的LifecycleOwner不一致的时候抛出异常。
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
        + " with different lifecycles");
    }
    //如果当前的ObserverWrapper不为空，则不进行任何操作。
    if (existing != null) {
        return;
    }
```
`LiveData#observe()`最后一句代码
```java
 owner.getLifecycle().addObserver(wrapper);
```
经过了上面分析，可知这句代码最后调用的地方为`LifecycleRegistry#addObserver(wrapper)`。
```java
    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

        if (previous != null) {
            return;
        }
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            // it is null we should be destroyed. Fallback quickly
            return;
        }

        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            // we do sync only on the top level.
            sync();
        }
        mAddingObserverCounter--;
    }
```
//TODO 未完



  [1]: https://www.youtube.com/watch?v=LmkKFCfmnhQ&t=42s
  [2]: https://developer.android.google.cn/topic/libraries/architecture/livedata
  [3]: https://developer.android.google.cn/topic/libraries/architecture/livedata
  [4]: https://developer.android.google.cn/reference/android/arch/lifecycle/MutableLiveData
  [5]: https://developer.android.google.cn/reference/android/arch/lifecycle/Transformations