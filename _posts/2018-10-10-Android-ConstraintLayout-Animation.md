---
layout: post
title:  "使用ConstraintLayout实现复杂动画"
date:   2018-10-10 14:42:54
categories: Android
tags: ConstraintLayout 
---

* content
{:toc}

本文记录如何使用ConstraintLayout实现复杂动画。





效果图：
![此处输入图片的描述][1]


```kotlin
class ConstraintDemoActivity : AppCompatActivity() {
    private var isAltLayout = false
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val originConstraintSet = ConstraintSet()
        val newConstraintSet = ConstraintSet()
        newConstraintSet.clone(this, R.layout.activity_constraint_demo_transform)

        setContentView(R.layout.activity_constraint_demo)
        val constraintLayout = findViewById<ConstraintLayout>(R.id.cst_demo)
        originConstraintSet.clone(constraintLayout)

        findViewById<ImageView>(R.id.iv_poster).setOnClickListener {
            val bounds = ChangeBounds()
            bounds.interpolator = LinearInterpolator()
            TransitionManager.beginDelayedTransition(constraintLayout, bounds)
            if (isAltLayout) {
                originConstraintSet.applyTo(constraintLayout)
            } else {
                newConstraintSet.applyTo(constraintLayout)
            }
            isAltLayout = !isAltLayout
        }
    }
}
```

> activity_constraint_demo
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/cst_demo"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".ConstraintDemoActivity">


    <ImageView
        android:id="@+id/iv_poster"
        android:layout_width="0dp"
        android:layout_height="300dp"
        android:layout_marginStart="8dp"
        android:layout_marginEnd="8dp"
        android:layout_marginBottom="8dp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintDimensionRatio="w,300:400"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:srcCompat="@drawable/poster" />

    <TextView
        android:id="@+id/tv_movie_title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginBottom="8dp"
        android:text="@string/movie_title"
        android:textColor="#333"
        android:textSize="16sp"
        app:layout_constraintBottom_toTopOf="@+id/iv_poster"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent" />

    <TextView
        android:id="@+id/tv_movie_desc"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginStart="8dp"
        android:layout_marginTop="8dp"
        android:layout_marginEnd="8dp"
        android:layout_marginBottom="8dp"
        android:text="@string/movie_desc"
        android:textColor="#888"
        android:textSize="14sp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/iv_poster"
        app:layout_constraintVertical_bias="0.0" />

    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:id="@+id/float_play"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginEnd="8dp"
        android:layout_marginBottom="8dp"
        android:scaleType="fitCenter"
        android:src="@drawable/ic_play"
        android:visibility="gone"
        app:fabSize="mini"
        app:layout_constraintBottom_toBottomOf="@+id/iv_poster"
        app:layout_constraintEnd_toEndOf="@+id/iv_poster" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

> activity_constraint_demo_transform
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/cst_demo"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".ConstraintDemoActivity">


    <ImageView
        android:id="@+id/iv_poster"
        android:layout_width="0dp"
        android:layout_height="170dp"
        android:layout_marginStart="8dp"
        android:layout_marginTop="8dp"
        android:layout_marginEnd="8dp"
        app:layout_constraintDimensionRatio="w,300:400"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:srcCompat="@drawable/poster" />

    <TextView
        android:id="@+id/tv_movie_title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="8dp"
        android:text="@string/movie_title"
        android:textColor="#333"
        android:textSize="16sp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/iv_poster" />

    <TextView
        android:id="@+id/tv_movie_desc"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginStart="8dp"
        android:layout_marginTop="8dp"
        android:layout_marginEnd="8dp"
        android:text="@string/movie_desc"
        android:textColor="#888"
        android:textSize="14sp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.0"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/tv_movie_title" />

    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:id="@+id/float_play"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginEnd="8dp"
        android:layout_marginBottom="8dp"
        app:fabSize="mini"
        android:scaleType="fitCenter"
        android:src="@drawable/ic_play"
        app:layout_constraintBottom_toBottomOf="@+id/iv_poster"
        app:layout_constraintEnd_toEndOf="@+id/iv_poster" />
    
</androidx.constraintlayout.widget.ConstraintLayout>
```


  [1]: http://qfxl.oss-cn-shanghai.aliyuncs.com/images/constraint_sample.gif?Expires=1539166274&OSSAccessKeyId=TMP.AQF0IFPkabUg1tLCYNbHO9apO3bygkaM-cZ3JgBtA_DUTDkgBgcWyX3jtN4tAAAAMCsCE1-lqzfv4n-WQOYtQX0bTjUpvesCFAy3BQ1YvGYT4xETJnfmjst67isc&Signature=lzYt/Sde9AdnwyO5fFlEV75M2j8=