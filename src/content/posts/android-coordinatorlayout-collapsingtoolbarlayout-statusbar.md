---
title: "CoordinatorLayout 与 CollapsingToolbarLayout 折叠时更改 StatusBar 的颜色"
pubDatetime: 2022-04-20T23:37:26+08:00
slug: android-coordinatorlayout-collapsingtoolbarlayout-statusbar
tags: ["Android", "CoordinatorLayout"]
description: "基于 AndroidX & Material Design"
---


![最终效果](/images/20220421-001516.gif)

基于 `AndroidX` & `Material Design` 

需求：在 `AppBarLayout` 展开时，`AppBarLayout` 中的布局延伸到 `StatusBar` 的下面，`StatusBar` 透明，`AppBarLayout` 折叠时，`StatusBar` 的颜色改为跟 `Toolbar` 一样的颜色

# 1. 布局

将需要延伸到 `StatusBar` 下的布局都加上 `android:fitsSystemWindows="true"`：

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true">

    <com.google.android.material.appbar.AppBarLayout
        android:id="@+id/app_bar"
        android:layout_width="match_parent"
        android:layout_height="256dp"
        android:fitsSystemWindows="true"
        android:theme="@style/Theme.Booster.AppBarOverlay">

        <com.google.android.material.appbar.CollapsingToolbarLayout
            style="@style/Widget.MaterialComponents.Toolbar.Primary"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:fitsSystemWindows="true"
            app:contentScrim="?attr/colorPrimary"
            app:layout_scrollFlags="scroll|exitUntilCollapsed"
            app:statusBarScrim="?attr/colorPrimary"
            app:toolbarId="@+id/toolbar">

            <ImageView
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:background="@android:color/holo_blue_bright"
                android:fitsSystemWindows="true" />

            <androidx.appcompat.widget.Toolbar
                android:id="@+id/toolbar"
                android:layout_width="match_parent"
                android:layout_height="?actionBarSize"
                app:layout_collapseMode="pin"
                app:popupTheme="@style/Theme.Booster.PopupOverlay" />

        </com.google.android.material.appbar.CollapsingToolbarLayout>

    </com.google.android.material.appbar.AppBarLayout>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"
        app:layout_behavior="com.google.android.material.appbar.AppBarLayout$ScrollingViewBehavior">

        <com.google.android.material.tabs.TabLayout
            android:id="@+id/tl_customer_detail"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:elevation="4dp"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent">

            <com.google.android.material.tabs.TabItem
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="@string/customer_detail_character" />

            <com.google.android.material.tabs.TabItem
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="@string/customer_detail_order" />

        </com.google.android.material.tabs.TabLayout>

        <androidx.viewpager2.widget.ViewPager2
            android:id="@+id/vp_customer_detail"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@id/tl_customer_detail" />

    </LinearLayout>

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

2. 主题

```xml
<style name="Theme.Booster.NoActionBar">
    <item name="android:statusBarColor">@android:color/transparent</item>
    <item name="windowActionBar">false</item>
    <item name="windowNoTitle">true</item>
</style>
```

3. 返回键

```kotlin
toolbar.setNavigationIcon(R.drawable.ic_round_arrow_white_24dp)
setSupportActionBar(toolbar)
supportActionBar?.run {
    setHomeButtonEnabled(true)
    setDisplayHomeAsUpEnabled(true)
}
```
