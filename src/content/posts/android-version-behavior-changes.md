---
title: "Android 各版本主要行为变更"
pubDatetime: 2024-08-29T13:54:52+08:00
slug: android-version-behavior-changes
tags: ["Android", "Framework"]
description: "Android 5.0 至 15 各版本的主要行为变更速查清单。"
---


### Android 15

- 屏幕录制检测
- [私密空间](https://developer.android.com/about/versions/15/features#private-space)
- 可以仅突出显示最近选择的照片 以及视频
- 数据同步前台服务超时行为 6 小时
- 引入对媒体文件进行转码等操作的服务类型
- 禁止与堆栈中的顶部 UID 不匹配的应用启动 activity

### Android 14

- 默认拒绝设定精确闹钟权限：不再向以 Android 13 及更高版本为目标平台的大多数新安装应用预先授予 `SCHEDULE_EXACT_ALARM` 权限，该权限默认处于拒绝状态
- 应用只能终止自己的后台进程：`killBackgroundProcesses()`
- 不能再安装 `targetSdkVersion` 低于 **23** 的应用
- [授予对照片和视频的部分访问权限](https://developer.android.com/about/versions/14/changes/partial-photo-video-access)
- 必须指定前台服务类型：`android:foregroundServiceType`

### Android 13 （API 33）

- 更新了 `Firebase Cloud Messaging (FCM)` 配额，从而提高了针对高优先级 FCM 显示通知的高优先级 FCM 传送的可靠性。
- 引入了运行时通知权限：`POST_NOTIFICATIONS`
- 将敏感内容复制到剪贴板时添加标志阻止敏感内容出现在内容预览中
- 停止使用共享用户 ID：`android:sharedUserId`
- 媒体权限细化：`READ_MEDIA_IMAGES`，`READ_MEDIA_VIDEO`，`READ_MEDIA_AUDIO`
- 在后台使用身体传感器需要新的权限：`BODY_SENSORS_BACKGROUND`
- 使用 Google Play 服务广告 ID 且以 Android 13（API 级别 33）及更高版本为目标平台的应用必须在其清单文件中声明常规 AD_ID权限：`<uses-permission android:name="com.google.android.gms.permission.AD_ID"/>`，否则会获取到一串 0

### Android 12 （API 31 – 32）

- Material You
- 如果应用请求 `ACCESS_FINE_LOCATION` 运行时权限，还应请求 `ACCESS_COARSE_LOCATION` 权限，以便处理用户授予应用大致位置访问权限的情形
- 当应用使用麦克风或相机时，图标会出现在状态栏中
- 当某个应用首次调用 [getPrimaryClip()](https://developer.android.com/reference/android/content/ClipboardManager?hl=zh-cn#getPrimaryClip()) 以从另一个应用访问剪辑数据时，会弹出一个消息框消息，通知用户对剪贴板的访问
- 应用无法关闭系统对话框：弃用了 `Intent.ACTION_CLOSE_SYSTEM_DIALOGS`
- Toast 上限为2行文本，并且必须在文本旁边显示应用图标
- 必须为使用 intent filter 的 Activity，Service，BroadcastReceiver 显式声明 `android:exported` 属性
- 精确闹钟权限：`SCHEDULE_EXACT_ALARM`
- 应用待机分桶 [Restricted Bucket](https://developer.android.com/topic/performance/appstandby#restricted-bucket)

### Android 11 （API 30）

- 单次授权
- 权限对话框 ”不再询问“
- 引入了数据访问审核：[AppOpsManager.OnOpNotedCallback](https://developer.android.com/reference/android/app/AppOpsManager.OnOpNotedCallback?hl=zh-cn)
- 限制 [getIccId()](https://developer.android.com/reference/android/telephony/SubscriptionInfo?hl=zh-cn#getIccId()) 方法访问不可重置的 ICCID，该方法会返回一个非 null 的空字符串。如需唯一标识设备上安装的 SIM 卡，请改用 [getSubscriptionId()](https://developer.android.com/reference/android/telephony/SubscriptionInfo?hl=zh-cn#getSubscriptionId()) 方法。订阅 ID 会提供一个索引值（从 1 开始），用于唯一识别已安装的 SIM 卡（包括实体 SIM 卡和电子 SIM 卡）。除非设备恢复出厂设置，否则此标识符的值对于给定 SIM 卡是保持不变的
- 软件包可见性过滤，意味着应用无法检测设备上安装的所有应用

### Android 10 （API 29）

- 支持可折叠设备：`resizeableActivity`
- 限制了应用程序在后台时的照片、影片和音乐档案的访问权限
- 限制了后台应用程序自动唤醒到前台
- 限制对IMEI码的读取

### Android 9 （API 28）

- 限制后台应用访问用户输入和传感器数据的能力
- 限制了对通话记录的访问权限
- 限制了对电话号码的访问权限
- 限制了对 WLAN 位置信息和连接信息的访问
- 限制使用非 SDK 接口
- 强制执行 FLAG_ACTIVITY_NEW_TASK，否则无法从非 activity 上下文启动 activity

### Android 8.0 （API 26 – 27）

- 后台执行限制，需要有前台服务
- 后台位置限制
- 引入了 安装来源不明的未知应用 权限
- 引入 通知渠道 `NotificationChannels`

### Android 7.0 （API 24 – 25）

- 后台优化：移除了三个常用的隐式广播 - `CONNECTIVITY_ACTION`、`ACTION_NEW_PICTURE` 和 `ACTION_NEW_VIDEO`
- 支持 Vulkan

### Android 6.0 （API 23）

- 全新的权限机制，针对 Android 6.0 及以上系统版本开发的应用程序在使用敏感权限（如拍照、查阅联系人或短信）时需要先征求用户同意
- 取消支持 Apache HTTP 客户端
- 移除对设备本地硬件标识符的编程访问权限， 使用 Wi-Fi 和 Bluetooth API 的应用。通过 [WifiInfo.getMacAddress()](https://developer.android.com/reference/android/net/wifi/WifiInfo?hl=zh-cn#getMacAddress()) 和 [BluetoothAdapter.getAddress()](https://developer.android.com/reference/android/bluetooth/BluetoothAdapter?hl=zh-cn#getAddress()) 方法 现在会返回常量值 `02:00:00:00:00:00`

### Android 5.0 （API 21 – 22）

- Material Design
- 支持64位处理器
- 全面由 Dalvik 虚拟机转用 [Android RunTime](https://zh.wikipedia.org/wiki/Android_Runtime "Android Runtime")（ART）编译虚拟机
