---
title: "使用 Jectpack Navigation 中 SafeArg 的方式向 Activity 与 Fragment 传递数据"
pubDatetime: 2022-09-25T23:14:45+08:00
slug: android-extra-kotlin-delegate
tags: ["Android", "Kotlin", "Jetpack Navigation"]
description: "在自己的项目中用 Jetpack 的 Navigation 的时候发现 SafeArgs 在 destination 之间传递参数非常符合我所期望的 Kotlin 该有的方式，所以我想把 SafeAr…"
---


在自己的项目中用 Jetpack 的 Navigation 的时候发现 SafeArgs 在 destination 之间传递参数非常符合我所期望的 Kotlin 该有的方式，所以我想把 SafeArgs 中传递参数的方式应用到 Activity 与 Fragment 中，并不想像现在使用 lateinit var 然后在 onCreate 等函数中去初始化这样用 Java 的方式去写 Kotlin，于是我开始尝试仿写一个 `activityArgs<T>()` 与 `fragmentArgs<T>()` 来在非 Navigation 的页面传递参数，所以先着手看 Jetpack Navigation SafeArgs 是如何使用代理传递参数的

# 1. Jetpack Navigation SafeArgs 是如何通过代理获取数据的
在 Navigation 组件中，SafeArgs 是通过在 navigation xml 中设置 `<argument>` 标签，然后通过 `navigation-safe-args-gradle-plugin` 插件解析xml来生成对应的数据类，这一过程可以略过不看，接着在 destination 中使用 `navArgs<T>()` 这个内联函数来获取传递过来的数据：
```kotlin file="DestinationFragment.kt"
private val args by navArgs<DestinationFragmentArgs>()
```

点进去查看 `navArgs<T>()` 的实现：
```kotlin file="ActivityNavArgsLazy.kt"
@MainThread
public inline fun <reified Args : NavArgs> Activity.navArgs(): NavArgsLazy<Args> =
    NavArgsLazy(Args::class) {
        intent?.let { intent ->
            intent.extras
                ?: throw IllegalStateException("Activity $this has null extras in $intent")
        } ?: throw IllegalStateException("Activity $this has a null Intent")
    }
```

重点就在这个 `NavArgsLazy` 类中，再点进去查看实现：
```kotlin file="NavArgsLazy.kt"
internal val methodSignature = arrayOf(Bundle::class.java)
internal val methodMap = ArrayMap<KClass<out NavArgs>, Method>()

public class NavArgsLazy<Args : NavArgs>(
    private val navArgsClass: KClass<Args>,
    private val argumentProducer: () -> Bundle
) : Lazy<Args> {
    private var cached: Args? = null

    override val value: Args
        get() {
            var args = cached
            if (args == null) {
                val arguments = argumentProducer()
                val method: Method = methodMap[navArgsClass]
                    ?: navArgsClass.java.getMethod("fromBundle", *methodSignature).also { method ->
                        // Save a reference to the method
                        methodMap[navArgsClass] = method
                    }
    
                @SuppressLint("BanUncheckedReflection") // needed for method.invoke
                @Suppress("UNCHECKED_CAST")
                args = method.invoke(null, arguments) as Args
                cached = args
            }
            return args
        }
    
    override fun isInitialized(): Boolean = cached != null
}
```

通过反射获取 `fromBundle` 方法，将 `argumentProducer` 传进来的 arguments (实际上就是 intent.extras 也就是 bundle) 通过调用反射方法的 `invoke` 函数传递进去生成 `args` 最后解析出来的对象

知道原理后就可以开始仿写：

# 2. activityArgs<T>()
定义接口，用来确定这个参数类是生成启动这个 Activity 的 `intent` 或者直接通过 `launch` 启动 Activity ：
```kotlin file="ActivityArgs.kt"
interface ActivityArgs {

    /**
     *  @return returns an intent that can be used to launch this activity.
     */
    fun intent(context: Context): Intent
    
    /**
     *  启动目标 Activity
     */
    fun launch(context: Context) = context.startActivity(intent(context))

}
```

然后定义一个接口，起到之前 `fromBundle` 函数的作用：
```kotlin file="ActivityArgsDeserializer.kt"
interface ActivityArgsDeserializer<T : ActivityArgs> {

    /**
     *  反序列化参数
     */
    fun deserialize(intent: Intent): T

}
```

委托类：
```kotlin file="ActivityArgsLazy.kt"
internal val activityArgsMethodSignature = arrayOf(Intent::class.java)
internal val activityArgsMethodMap = ArrayMap<KClass<out ActivityArgs>, Method>()

class ActivityArgsLazy<Args : ActivityArgs>(
    private val activityArgsClass: KClass<Args>,
    private val intentProducer: () -> Intent
) : Lazy<Args> {

    private var cached: Args? = null
    
    override val value: Args
        get() {
            var args = cached
            if (args == null) {
                val intent = intentProducer()
                val method: Method = activityArgsMethodMap[activityArgsClass]
                    ?: activityArgsClass.java.getMethod("deserialize", *activityArgsMethodSignature)
                        .also { method ->
                            // Save a reference to the method
                            activityArgsMethodMap[activityArgsClass] = method
                        }
    
                @SuppressLint("BanUncheckedReflection") // needed for method.invoke
                @Suppress("UNCHECKED_CAST")
                args = method.invoke(null, intent) as Args
                cached = args
            }
            return args
        }
    
    override fun isInitialized(): Boolean = cached != null
}
```

使用：
```kotlin file="MainActivityArgs.kt"
data class MainActivityArgs(
    val text: String
) : ActivityArgs {
    override fun intent(context: Context): Intent =
        Intent(context, MainActivity::class.java).apply {
            putExtra("TEST_TEXT", text)
        }

    companion object : ActivityArgsDeserializer<MainActivityArgs> {
    
        @JvmStatic
        override fun deserialize(intent: Intent): MainActivityArgs = intent.run {
            MainActivityArgs(
                getStringExtra("TEST_TEXT")
            )
        }
    }
}
```

调用方：
```kotlin
MainActivityArgs("Test").launch(this)
```

接收方：
```kotlin file="MainActivity.kt"
private val args by activityArgs<MainActivityArgs>()
```

# 3. fragmentArgs<T>()
类似 `activityArgs<T>()`，先创建接口：
```kotlin file="FragmentArgs.kt"
interface FragmentArgs {

    /**
     * @return returns a new instance of that fragment
     */
    fun newInstance(): Fragment

}
```

接着定义解析方法接口：
```kotlin file="FragmentArgsDeserializer.kt"
interface FragmentArgsDeserializer<T : FragmentArgs> {

    fun deserialize(arguments: Bundle): T

}
```

委托类：
```kotlin file="FragmentArgsLazy.kt"
internal val fragmentMethodSignature = arrayOf(Bundle::class.java)
internal val fragmentMethodMap = ArrayMap<KClass<out FragmentArgs>, Method>()

class FragmentArgsLazy<Args : FragmentArgs>(
    private val fragmentArgsClass: KClass<Args>,
    private val bundleProducer: () -> Bundle
) : Lazy<Args> {

    private var cached: Args? = null

    override val value: Args
        get() {
            var args = cached
            if (args == null) {
                val bundle = bundleProducer()
                val method: Method = fragmentMethodMap[fragmentArgsClass]
                    ?: fragmentArgsClass.java.getMethod("deserialize", *fragmentMethodSignature)
                        .also { method ->
                            // Save a reference to the method
                            fragmentMethodMap[fragmentArgsClass] = method
                        }

                @SuppressLint("BanUncheckedReflection") // needed for method.invoke
                @Suppress("UNCHECKED_CAST")
                args = method.invoke(null, bundle) as Args
                cached = args
            }
            return args
        }

    override fun isInitialized(): Boolean = cached != null
}
```

使用：
```kotlin file="MainFragmentArgs.kt"
data class MainFragmentArgs(
    val text: String
) : FragmentArgs {
    override fun newInstance(): MainFragment =
        MainFragment().apply {
            arguments = Bundle().apply {
                putString("TEST_TEXT", text)
            }
        }

    companion object : FragmentArgsDeserializer<MainFragmentArgs> {
    
        @JvmStatic
        override fun deserialize(bundle: Bundle): MainFragmentArgs = bundle.run {
            MainFragmentArgs(
                getString("TEST_TEXT")
            )
        }
    }
}
```

调用方：
```kotlin
MainFragmentArgs("test").newInstance()
```

接收方：
```kotlin file="MainFragment.kt"
private val args by fragmentArgs<MainFragmentArgs>()
```
