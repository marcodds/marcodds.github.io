---
layout: post
title:  "使用ActivityLifecycleCallbacks管理Activity"
date:   2018-09-01 16:43:54
categories: Android
tags: ActivityLifecycleCallbacks 
---

* content
{:toc}

APP的开发过程中有时候需要自己去管理Activity，本文记录如何使用[ActivityLifecycleCallbacks][1]来实现管理Activity的需求。




# 使用ActivityLifecycleCallbacks管理Activity

转载请注明出处：https://blog.csdn.net/xuyonghong1122

APP的开发过程中有时候需要自己去管理Activity，本文记录如何使用[ActivityLifecycleCallbacks][1]来实现管理Activity的需求。

## 什么是ActivityLifecycleCallbacks？
[ActivityLifecycleCallbacks][2]是`Application`下的一个接口，位于
```java
public class Application extends ContextWrapper implements ComponentCallbacks2 {
    private ArrayList<ComponentCallbacks> mComponentCallbacks =
            new ArrayList<ComponentCallbacks>();
    private ArrayList<ActivityLifecycleCallbacks> mActivityLifecycleCallbacks =
            new ArrayList<ActivityLifecycleCallbacks>();
    private ArrayList<OnProvideAssistDataListener> mAssistCallbacks = null;

    /** @hide */
    public LoadedApk mLoadedApk;

    public interface ActivityLifecycleCallbacks {
        void onActivityCreated(Activity activity, Bundle savedInstanceState);
        void onActivityStarted(Activity activity);
        void onActivityResumed(Activity activity);
        void onActivityPaused(Activity activity);
        void onActivityStopped(Activity activity);
        void onActivitySaveInstanceState(Activity activity, Bundle outState);
        void onActivityDestroyed(Activity activity);
    }
```

## 如何使用ActivityLifecycleCallbacks监听Activity的声明周期？
项目的`Application`实现`ActivityLifecycleCallbacks`接口并调用`registerActivityLifecycleCallbacks`：
```kotlin
class App : Application(), Application.ActivityLifecycleCallbacks {
    override fun onCreate() {
        super.onCreate()
        registerActivityLifecycleCallbacks(this)
    }
    override fun onActivityPaused(activity: Activity?) {
        "onActivityPaused".log()
    }

    override fun onActivityResumed(activity: Activity?) {
        "onActivityResumed".log()
    }

    override fun onActivityStarted(activity: Activity?) {
        "onActivityStarted".log()
    }

    override fun onActivityDestroyed(activity: Activity?) {
        "onActivityDestroyed".log()
    }

    override fun onActivitySaveInstanceState(activity: Activity?, outState: Bundle?) {
        "onActivitySaveInstanceState".log()
    }

    override fun onActivityStopped(activity: Activity?) {
        "onActivityStopped".log()
    }

    override fun onActivityCreated(activity: Activity?, savedInstanceState: Bundle?) {
        "onActivityCreated".log()
    }

    private fun String.log() {
        Log.i("qfxl", this)
    }
}
```
声明项目的`Application`
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.qfxl.sample">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:name=".App"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```
启动Activity
打印log
>   onActivityCreated
    onActivityStarted
    onActivityResumed

关闭Activity
>   onActivityPaused
    onActivityStopped
    onActivityDestroyed
    
## ActivityLifecycleCallbacks的原理。
从`Application`里的`registerActivityLifecycleCallbacks`入手。
```java
   public void registerActivityLifecycleCallbacks(ActivityLifecycleCallbacks callback) {
        synchronized (mActivityLifecycleCallbacks) {
            mActivityLifecycleCallbacks.add(callback);
        }
    }
```
`mActivityLifecycleCallbacks`是`ArrayList<ActivityLifecycleCallbacks>`在调用了`registerActivityLifecycleCallbacks`之后会往里面添加`registerActivityLifecycleCallbacks`的实现。

既然是监听`Activity`的生命周期，那在`Activity`的各个声明周期方法里肯定有相关代码，这里以`Activity`的`onCreate`为例。
```java
protected void onCreate(@Nullable Bundle savedInstanceState) {
        if (DEBUG_LIFECYCLE) Slog.v(TAG, "onCreate " + this + ": " + savedInstanceState);

        if (getApplicationInfo().targetSdkVersion >= O_MR1 && mActivityInfo.isFixedOrientation()) {
            final TypedArray ta = obtainStyledAttributes(com.android.internal.R.styleable.Window);
            final boolean isTranslucentOrFloating = ActivityInfo.isTranslucentOrFloating(ta);
            ta.recycle();

            if (isTranslucentOrFloating) {
                throw new IllegalStateException(
                        "Only fullscreen opaque activities can request orientation");
            }
        }

        if (mLastNonConfigurationInstances != null) {
            mFragments.restoreLoaderNonConfig(mLastNonConfigurationInstances.loaders);
        }
        if (mActivityInfo.parentActivityName != null) {
            if (mActionBar == null) {
                mEnableDefaultActionBarUp = true;
            } else {
                mActionBar.setDefaultDisplayHomeAsUpEnabled(true);
            }
        }
        if (savedInstanceState != null) {
            mAutoFillResetNeeded = savedInstanceState.getBoolean(AUTOFILL_RESET_NEEDED, false);
            mLastAutofillId = savedInstanceState.getInt(LAST_AUTOFILL_ID,
                    View.LAST_APP_AUTOFILL_ID);

            if (mAutoFillResetNeeded) {
                getAutofillManager().onCreate(savedInstanceState);
            }

            Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);
            mFragments.restoreAllState(p, mLastNonConfigurationInstances != null
                    ? mLastNonConfigurationInstances.fragments : null);
        }
        mFragments.dispatchCreate();
        //这里回调了Application的dispatchActivityCreated
        getApplication().dispatchActivityCreated(this, savedInstanceState);
        if (mVoiceInteractor != null) {
            mVoiceInteractor.attachActivity(this);
        }
        mRestoredFromBundle = savedInstanceState != null;
        mCalled = true;
    }
```
在`Activity`的`onCreate`方法中有一句代码
```java
getApplication().dispatchActivityCreated(this, savedInstanceState);
```
`getApplication`获取的是当前应用的`Application`对象，具体的可以查看`ActivityThread`的`performLaunchActivity`，或者在我之前记录的[Activity启动流程文章中查看][3]，这里不赘述。

点开`Application`的`dispatchActivityCreated`。
```java
   /* package */ void dispatchActivityCreated(Activity activity, Bundle savedInstanceState) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i=0; i<callbacks.length; i++) {
                ((ActivityLifecycleCallbacks)callbacks[i]).onActivityCreated(activity,
                        savedInstanceState);
            }
        }
    }
```
该方法中会循环调用`mActivityLifecycleCallbacks`中的所有实现了`ActivityLifecycleCallbacks`的`onActivityCreated`方法。
至此，原理分析完毕。

## 如何利用ActivityLifecycleCallbacks来实现Activity的管理。
记录一下我常用的管理方式。
```kotlin
object ActivityPool : Application.ActivityLifecycleCallbacks {

    private val activityStack = Stack<Activity>()

    fun init(app: Application) {
        app.registerActivityLifecycleCallbacks(this)
    }

    override fun onActivityPaused(activity: Activity?) {

    }

    override fun onActivityResumed(activity: Activity?) {

    }

    override fun onActivityStarted(activity: Activity?) {

    }

    override fun onActivitySaveInstanceState(activity: Activity?, outState: Bundle?) {

    }

    override fun onActivityStopped(activity: Activity?) {

    }

    override fun onActivityDestroyed(activity: Activity?) {
        activityStack.remove(activity)
    }

    override fun onActivityCreated(activity: Activity?, savedInstanceState: Bundle?) {
        activityStack.add(activity)
    }

    /**
     * 获取倒数第二个 Activity
     *
     * @return
     */
    fun getPenultimateActivity(currentActivity: Activity): Activity? {
        var activity: Activity? = null
        try {
            if (activityStack.size > 1) {
                activity = activityStack[activityStack.size - 2]

                if (currentActivity == activity) {
                    val index = activityStack.indexOf(currentActivity)
                    if (index > 0) {
                        // 处理内存泄漏或最后一个 Activity 正在 finishing 的情况
                        activity = activityStack[index - 1]
                    } else if (activityStack.size == 2) {
                        // 处理屏幕旋转后 activityStack 中顺序错乱
                        activity = activityStack.lastElement()
                    }
                }
            }
        } catch (e: Exception) {
        }
        return activity
    }

    /**
     * 获取栈底Activity
     */
    fun getLastActivity() = activityStack.lastElement()

    /**
     * 获取栈顶Activity
     */
    fun getCurrentActivity() = activityStack.firstElement()

    /**
     * 数量
     */
    fun size() = activityStack.size
}
```
```kotlin
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        //管理Activity
        ActivityPool.init(this)
    }
}
```
关于`ActivityLifecycleCallbacks`的记录完毕。






  [1]: https://developer.android.google.cn/reference/android/app/Application.ActivityLifecycleCallbacks.html#onActivityCreated%28android.app.Activity,%20android.os.Bundle%29
  [2]: https://developer.android.google.cn/reference/android/app/Application.ActivityLifecycleCallbacks.html#onActivityCreated%28android.app.Activity,%20android.os.Bundle%29
  [3]: https://blog.csdn.net/xuyonghong1122/article/details/79600038