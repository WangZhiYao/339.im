---
title: "MVI 架构 Android 开发的设计与实践指南"
pubDatetime: 2025-09-01T23:19:09+08:00
slug: mvi-architecture
tags: ["Android", "MVI", "Architecture"]
description: "在现代应用开发中，我们经常面临构建状态密集型应用的挑战，例如实时交易仪表盘、物联网设备控制面板或复杂的社交媒体客户端。这类应用的共同特点是："
---


## 1. 引言：为何需要现代化的架构？

### 1.1 应用开发的挑战

在现代应用开发中，我们经常面临构建**状态密集型应用**的挑战，例如实时交易仪表盘、物联网设备控制面板或复杂的社交媒体客户端。这类应用的共同特点是：
* **状态密集型UI**：需要实时、高频地展示来自多个数据源的状态信息，如网络状态、传感器数据、用户配置、后端推送等。
* **复杂的交互逻辑**：用户的交互行为需要触发一系列复杂的业务逻辑或向后端发送指令。
* **高可靠性要求**：状态的展示必须精准、实时，用户的操作与系统的反馈流程必须清晰、可追溯，以确保应用的稳定性和可靠性。

在这样的背景下，我们需要一个能够清晰、健壮地管理复杂状态和用户交互的架构模式。

### 1.2 为什么选择 MVI 架构？

传统的 MVVM 模式在处理简单页面时表现良好，但在我们这种状态复杂、交互频繁的场景下，MVI（Model-View-Intent）展现出更明显的优势：
* **可预测的状态管理**：MVI 通过其核心的**单向数据流**原则，确保了 UI 状态的每一次变更都是可预测且可追溯的。这对于调试因多源数据并发更新而导致的 UI 异常问题至关重要。
* **统一的状态容器**：MVI 将整个页面的 UI 状态聚合在单一的 `UiState` 对象中。这就像一个集中的仪表盘，让我们可以一目了然地看到界面的完整快照，极大地简化了对复杂状态的理解和管理。
* **清晰的意图驱动**：用户的每一个操作（如点击按钮）都被抽象为一个明确的“意图”（Intent）。这种模式使得业务逻辑的触发点非常清晰，与指令驱动的交互模型天然契合。

### 1.3 MVI 与 MVVM 的核心差异对比

为了更具体地说明 MVI 的优势，我们将其与大家熟悉的 MVVM 模式进行对比：

| 特性         | MVI (Model-View-Intent)                                   | MVVM (Model-View-ViewModel)                         | 在复杂应用中的优势                                                   |
| ---------- | --------------------------------------------------------- | --------------------------------------------------- | ----------------------------------------------------------- |
| **状态管理**   | **单一 `UiState` 对象**，包含所有UI所需数据。                           | **多个 `LiveData` / `StateFlow` 对象**，分散管理不同状态。        | **更集中**：面对几十个乃至上百个状态值，单一State对象让数据流更清晰，避免了管理大量LiveData订阅的复杂性。 |
| **状态更新**   | 通过 `reduce` 函数，基于旧 State 创建**全新的 State 对象**。              | 直接修改 `LiveData` / `StateFlow` 的 `value`。            | **更安全**：不可变性设计从根本上避免了多线程并发修改状态的风险，状态变更原子化，易于追踪和调试。          |
| **新增状态成本** | 在 `UiState` data class 中**增加一个字段**。                       | **新增一个 `StateFlow` 实例**及相应的 `MutableStateFlow`。     | **开销更小**：MVI 的方式更轻量，减少了模板代码和 ViewModel 中的对象数量。              |
| **数据流向**   | **严格的单向循环**：View -> Intent -> ViewModel -> State -> View。 | **双向绑定**（DataBinding）或**单向数据流**（ViewModel -> View）。 | **更可预测**：严格的单向数据流使得任何状态问题的排查路径都非常清晰，总能回溯到触发它的那个 Intent。     |
| **一次性事件**  | 内置 `SideEffect` (副作用) 概念，通过 `SharedFlow` 处理。              | 通常通过 `SharedFlow` 或自定义的 `Event` 包装类实现。              | **概念更清晰**：MVI 将“状态”和“一次性事件”明确分离，架构层面就定义了如何处理 Toast、导航等事件。   |

**结论**：对于需要高度关注**状态一致性**和**行为可追溯性**的应用，MVI 提供的单一状态容器和严格的单向数据流，相比 MVVM 能带来更强的可维护性、可预测性和更低的调试成本。

## 2. MVI 核心理念

### 2.1 核心思想

MVI 将应用逻辑清晰地划分为三个核心部分，构成一个循环的数据流：
* **Model**: 代表UI的**状态 (State)**。在本框架中，它是一个不可变的数据类，由 ViewModel 中的 `StateFlow` 持有。它是UI渲染的唯一数据源 (Single Source of Truth)。
* **View**: 负责渲染 State 并将用户的操作转化为**意图 (Intent)** 发送给 ViewModel。
* **Intent**: 代表用户的操作或应用的事件（如点击、输入）。它会触发 ViewModel 中的业务逻辑。

### 2.2 单向数据流图

这种单向数据流确保了状态的变更总是可追溯的，极大地简化了调试和复杂状态的管理。

![MVI Architecture](/images/mvi-architecture.svg)

## 3. 框架核心组件解析

### 3.1 核心结构设计

![MVI 核心结构](/images/mvi-architecture-class.svg)

### 3.2 MVIContainer<STATE, SIDE_EFFECT> 接口

此接口定义了 MVI 结构的核心契约。任何 ViewModel 都需要实现它，以暴露状态和副作用的输出通道。
* **uiState: StateFlow\<STATE\>**
  * **用途**: 一个只读的、热的 Flow，持有当前界面的完整状态。
  * **特性**: `StateFlow` 保证了订阅者总能立即收到最新的状态值（conflation），并且在屏幕旋转等配置变更后能恢复最后的状态，非常适合用于表示 UI State。
* **sideEffect: SharedFlow\<SIDE_EFFECT\>**
  * **用途**: 一个只读的、热的 Flow，用于处理**一次性事件 (One-time Events)**。
  * **定义**: 副作用是指那些不属于UI状态本身的事件，例如：弹出 Toast、导航到新页面、显示 SnackBar 等。这些事件被消费后就不应再次触发（例如，屏幕旋转后不应再次弹出 Toast）。
  * **特性**: `SharedFlow` 能够确保每个事件只被下游的一个或多个订阅者消费一次，非常适合用于处理副作用。
* **reduce(reducer: (STATE) -> STATE)**:
  * **更新状态的唯一入口**。它接收一个 reducer lambda，该 lambda 基于当前状态 (`state`) 计算并返回一个新状态。
  * 这种设计强制使用 `state.copy()` 方法创建新状态对象，保证了状态的**不可变性**，使得状态变更原子化且易于追踪。
* **postSideEffect(sideEffect: SIDE_EFFECT)**:
  * **发送一次性事件的唯一入口**。在协程中调用此方法来触发一个副作用。

### 3.3 BaseMVIViewModel / BaseAndroidMVIViewModel

这是 ViewModel 的抽象基类，封装了 MVI 的通用逻辑，开发者只需关注业务本身。
* **状态管理**:
  * 内部持有 `_uiState: MutableStateFlow`，对外仅暴露不可变的 `uiState: StateFlow`。这确保了状态的修改权被严格控制在 ViewModel 内部，符合单一数据源原则。
* **副作用管理**:
  * 内部持有 `_sideEffect: MutableSharedFlow`，对外暴露不可变的 `sideEffect: SharedFlow`。

### 3.4 observe 扩展函数

这是一个连接 `MVIContainer` (ViewModel) 和 `LifecycleOwner` (Activity/Fragment) 的工具函数。

```kotlin file="MVIContainerExt.kt"
fun <STATE : Any, SIDE_EFFECT : Any> MVIContainer<STATE, SIDE_EFFECT>.observe(
    lifecycleOwner: LifecycleOwner,
    lifecycleState: Lifecycle.State = Lifecycle.State.STARTED,
    state: (suspend (state: STATE) -> Unit)? = null,
    sideEffect: (suspend (sideEffect: SIDE_EFFECT) -> Unit)? = null
) {
    lifecycleOwner.lifecycleScope.launch {
        lifecycleOwner.lifecycle.repeatOnLifecycle(lifecycleState) {
            state?.let { launch { uiState.collect { state(it) } } }
            sideEffect?.let { launch { this@observe.sideEffect.collect { sideEffect(it) } } }
        }
    }
}
```

* **生命周期安全**:
  * 内部使用 `lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED)` 来启动协程并收集 Flow。
  * 这能确保：
    * 当 View 进入 `STARTED` 状态时，开始收集 `uiState` 和 `sideEffect`。
    * 当 View 进入 `STOPPED` 状态时，自动取消协程，停止收集，从而防止内存泄漏和在后台更新UI。
* **简化模板代码**:
  * 将订阅 `StateFlow` 和 `SharedFlow` 的样板代码封装起来，让 UI 层的逻辑更加简洁和聚焦于核心职责：`render` 和 `handleSideEffect`。

## 4. 框架使用指南：以一个典型功能为例

### 4.1 步骤一：定义 UiState 和 SideEffect

* `State` 必须是 `data class`，以利用其 `copy()` 方法和 `equals()` 实现，方便状态更新和 UI diff。
* `SideEffect` 推荐使用 `sealed class`，这样 `when` 语句可以进行详尽性检查，确保所有可能的副作用都被处理。

```kotlin
// 定义UI状态：包含所有渲染UI所需的数据
data class FeatureUiState(
    val isLoading: Boolean = false,
    val inputText: String = "",
    val resultData: String? = null
)

// 定义副作用：所有一次性事件的集合
sealed class FeatureSideEffect {
    data class ShowToast(val message: String) : FeatureSideEffect()
    object OperationSuccess : FeatureSideEffect()
    object NavigateToNextScreen : FeatureSideEffect()
}
```

### 4.2 步骤二：创建 ViewModel

继承 `BaseMVIViewModel`，将用户的操作（Intent）实现为 ViewModel 的公共方法。在这些方法中处理业务逻辑，并通过 `reduce` 更新状态或通过 `postSideEffect` 发送副作用。

```kotlin file="FeatureViewModel.kt"
class FeatureViewModel : BaseMVIViewModel<FeatureUiState, FeatureSideEffect>() {
    // 定义初始状态
    override val initState: FeatureUiState
        get() = FeatureUiState()

    /**
     * Intent: 用户点击了提交按钮
     */
    fun onSubmitClicked() {
        viewModelScope.launch {
            // 1. 更新UI为加载状态
            reduce { it.copy(isLoading = true) }
            
            // 2. 执行业务逻辑（如网络请求）
            val success = someSuspendApiCall(currentState.inputText)
            
            // 3. 根据结果更新状态并发送副作用
            if (success) {
                reduce { it.copy(isLoading = false, resultData = "Success!") }
                postSideEffect(FeatureSideEffect.OperationSuccess)
            } else {
                reduce { it.copy(isLoading = false) }
                postSideEffect(FeatureSideEffect.ShowToast("Operation Failed"))
            }
        }
    }

    /**
     * Intent: 用户输入了文本
     */
    fun onInputTextChanged(text: String) {
        // 通过 reduce 原子化地更新状态
        reduce { state -> state.copy(inputText = text) }
    }

    // ... 其他 Intents
}
```

### 4.3 步骤三：在 View 中观察数据

在 Activity 或 Fragment 中，使用 `observe` 扩展函数将 UI 与 ViewModel 连接起来。

```kotlin file="FeatureFragment.kt"
class FeatureFragment : BaseBindingFragment<FeatureFragmentBinding>(FeatureFragmentBinding::inflate) {
    
    private val viewModel by viewModels<FeatureViewModel>()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // 将ViewModel的输出连接到View层的渲染和处理函数
        viewModel.observe(
            lifecycleOwner = viewLifecycleOwner,
            state = ::render,
            sideEffect = ::handleSideEffect
        )
        
        // 设置点击监听，触发ViewModel的Intent
        binding.submitButton.setOnClickListener {
            viewModel.onSubmitClicked()
        }
    }

    /**
     * 根据最新的 uiState 渲染界面
     */
    private fun render(uiState: FeatureUiState) {
        // 更新UI组件，例如：
        binding.progressBar.isVisible = uiState.isLoading
        binding.submitButton.isEnabled = !uiState.isLoading
        binding.resultTextView.text = uiState.resultData
    }

    /**
     * 处理一次性副作用事件
     */
    private fun handleSideEffect(sideEffect: FeatureSideEffect) {
        when (sideEffect) {
            is FeatureSideEffect.ShowToast -> Toast.makeText(context, sideEffect.message, Toast.LENGTH_SHORT).show()
            is FeatureSideEffect.OperationSuccess -> {
                // 处理成功事件
            }
            is FeatureSideEffect.NavigateToNextScreen -> {
                // 执行导航
            }
        }
    }
}
```

## 5. 核心设计原则与收益

* **单一数据源**
  * UI 的所有渲染都仅依赖于 `UiState`。这消除了多个状态源导致的数据不一致问题，使UI状态变得高度可预测。
* **状态不可变性**
  * `UiState` 是一个不可变对象 (`data class`)。任何状态的变更都通过 `copy()` 创建一个全新的对象。这避免了多线程并发修改数据的风险，也使得状态变更记录和调试成为可能。
* **清晰的职责分离**
  * **ViewModel**: 纯粹的业务逻辑中心。它接收 Intent，处理数据，并输出 State 和 SideEffect。它完全独立于 Android 框架，不持有任何 View 的引用。
  * **View (Fragment/Activity)**: 哑视图 (Dumb View)。它只负责渲染 State 和发送用户 Intent，不包含任何业务逻辑。
* **出色的可测试性**
  * 由于 ViewModel 的职责单一且无依赖 Android 框架，可以非常方便地进行单元测试。测试逻辑通常遵循 "Given-When-Then" 模式：
    * **Given**: 一个处于特定初始状态的 ViewModel。
    * **When**: 调用一个 Intent 方法。
    * **Then**: 断言 ViewModel 的 `uiState` 是否迁移到了预期的新状态，以及是否发送了正确的 `sideEffect`。
* **内置生命周期安全**
  * 通过 `repeatOnLifecycle` 机制，框架从根本上解决了协程泄漏和在后台更新UI的问题，开发者无需手动管理协程的启动与取消。

## 6. 设计决策与思考

本节将阐述框架在设计过程中做出的一些关键决策及其背后的考量，这有助于更深入地理解框架的设计思路。

### 6.1. 关于 Intent 设计的思考：公共方法 vs. Sealed Class

在 MVI 架构中，如何定义“意图 (Intent)”通常有两种主流方式。本框架选择了将 ViewModel 的公共方法作为 Intent，这与另一种常见的 Sealed Class 模式形成了对比。我们认为，在大多数 Android 开发场景下，公共方法是更具优势的选择。

**1. 核心差异对比**

| **特性** | **方案一：公共方法 (本框架采用)** | **方案二：Sealed Class** |
|---|---|---|
| **View层调用** | 直观的方法调用，符合常规编程习惯。 | 需要先创建Intent对象，再通过统一方法分发。 |
| **ViewModel实现** | 逻辑分散在不同的小方法中，职责单一。 | 逻辑集中在单一方法的when语句中，可能变得臃肿。 |
| **心智模型** | 简单、直接。 | 增加了Intent流转的抽象层，需要考虑缓冲、背压等。 |

**2. 代码示例对比**

**场景**：用户在输入框中改变了文本。

**方案一：公共方法 (本框架采用)**
```kotlin
// View 层调用，简洁直观
binding.editText.doOnTextChanged { text, _, _, _ ->
    viewModel.onInputTextChanged(text.toString())
}

// ViewModel 实现，职责清晰
class FeatureViewModel : BaseMVIViewModel<...> {
    fun onInputTextChanged(value: String) {
        reduce { state -> state.copy(inputText = value) }
    }
    // ... 其他方法
}
```

**方案二：Sealed Class**
```kotlin
// View 层调用，略显繁琐
binding.editText.doOnTextChanged { text, _, _, _ ->
    val intent = FeatureIntent.TextChanged(text.toString())
    viewModel.accept(intent)
}

// ViewModel 实现，逻辑集中
class FeatureViewModel : ViewModel() {
    private val store = object : BaseMVIStore<FeatureIntent, ...> {
        override suspend fun onIntent(intent: FeatureIntent) {
            when (intent) {
                is FeatureIntent.TextChanged -> {
                    setState { copy(inputText = intent.value) }
                }
                // ... 其他 when 分支
            }
        }
    }
    fun accept(intent: FeatureIntent) = store.accept(intent)
}
```

**3. 设计考量**

Sealed Class 方案在理论上提供了一个统一的意图入口，追求 MVI 模式的理论完整性，但在实践中，公共方法方案的优势更为突出：

* **开发效率**：公共方法对 IDE 的自动补全功能非常友好，代码可发现性强，能有效提升开发效率。
* **简洁性**：避免了为每个操作都定义一个数据类和编写 when 分支的样板代码，尤其是在交互复杂的页面，这一点尤为重要。
*   **可读性**：将每个意图的处理逻辑封装在独立的方法中，比维护一个可能长达数百行的 when 语句块更易于阅读和维护。
*   **与主流框架实践保持一致**：许多成熟的 Android MVI 框架，如 **Orbit MVI** 和 **Mavericks (by Airbnb)**，都倾向于将 ViewModel 的公共方法作为用户意图的入口。这种设计模式在业界得到了广泛验证，证明了其在实际项目中的高效性和可维护性。这表明，将公共方法作为 Intent 并非是“非主流”或“不纯粹”的做法，反而是被广泛接受和推崇的实践。

因此，我们选择了一个更务实、更符合 Android 开发者直觉的方案。

### 6.2 架构分层思考：集成式 ViewModel vs. 独立 Store

本框架的设计思路是“ViewModel IS A Store”，通过继承将状态管理能力融入 BaseMVIViewModel。这与另一种“ViewModel HAS A Store”的组合模式（即 ViewModel 持有一个独立的 Store 实例）有所不同。

**1. 核心差异对比**

| **特性** | **方案一：集成式 ViewModel (本框架采用)** | **方案二：独立 Store** |
|---|---|---|
| **实现方式** | 继承 BaseMVIViewModel | 在 ViewModel 中组合一个 Store 实例 |
| **代码结构** | 简洁，无冗余的委托代码。 | 存在较多模板代码，如 ViewModel 转发 Store 的属性和方法。 |
| **可测试性** | ViewModel 本身就是业务逻辑单元，易于测试。 | 理论上可单独测试Store，但错误的实现（如匿名内部类）会使其难以测试。 |

**2. 代码示例对比**

**方案一：集成式 ViewModel (本框架采用)**
```kotlin file="FeatureFragment.kt"
// ViewModel 定义，简洁明了
class FeatureViewModel : BaseMVIViewModel<FeatureUiState, FeatureSideEffect>() {
    
    override val initState = FeatureUiState()

    fun onSubmitClicked() {
        // 直接调用基类方法
        reduce { state -> state.copy(...) }
    }
}
```

**方案二：独立 Store (以匿名内部类方式实现)**
```kotlin
sealed class FeatureIntent {
    object SubmitClicked
    // ...
}

// ViewModel 定义，结构复杂，充满模板代码
class FeatureViewModel : ViewModel() {
    // 1. 样板代码：创建 Store 实例
    private val store = object : BaseMVIStore<FeatureIntent, FeatureUiState, FeatureSideEffect>(...) {
        override suspend fun onIntent(intent: FeatureIntent) {
            when (intent) {
                // 2. 业务逻辑深嵌在匿名内部类中
                is SubmitClicked -> { setState { copy(...) } }
                // ...
            }
        }
    }

    // 3. 样板代码：对外暴露 Store 的属性和方法
    val state = store.state
    val effects = store.effects
    fun accept(intent: ...) = store.accept(intent)
}
```

**3. 设计考量**
虽然独立的 Store 模式在理论上追求更彻底的职责分离，但在 Android 实践中，集成式方案的优势更加明显：

* **拥抱 Android 生态**：我们的方案充分利用了 Jetpack ViewModel 的设计初衷，即作为生命周期感知的业务逻辑处理单元。独立的 Store 模式在某种程度上是重复造轮子，并架空了 ViewModel 的核心作用。
* **避免实现陷阱**：如代码示例所示，独立的 Store 模式很容易因不当实现（如使用匿名内部类）而导致可测试性这一理论优势完全丧失，同时还带来了更差的可读性。
* **提升代码简洁度**：通过继承，我们消除了所有不必要的转发代码和复杂的内部结构，让开发者能更专注于业务本身。

综上，集成式 ViewModel 方案，是在 MVI 思想和 Android 平台特性之间取得的一个最佳平衡，它更简单、更健壮，也更易于维护和测试。

### 6.3 优缺点对比分析

| 对比维度   | 方案 A：集成式 ViewModel (本框架)                                                                                                                                                          | 方案 B：独立 Store 模式                                                                                                                                                                      |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **优点** | 简洁、直观：代码量少，没有不必要的封装和转发，符合 Android 开发直觉。<br><br>开发效率高：通过 IDE 代码补全，ViewModel 的所有可用操作。<br><br>易于测试：ViewModel 本身就是业务逻辑单元，可以直接实例化进行单元测试，测试代码编写简单。<br><br>线程安全：将线程调度器交由开发者自主设置，模型简单且安全。 | 理论上职责分离更彻底：将状态管理逻辑完全从 ViewModel 中剥离，理论上可以实现业务逻辑的跨平台复用（KMP）。<br><br>统一的意图入口：所有 Intent 都经过 accept 方法，便于在此处进行集中的日志记录、埋点等切面操作。                                                            |
| **缺点** | 切面处理需额外设计：如果需要对所有 Intent 进行统一处理（例如日志），需要自行添加。                                                                                                                                     | 实现复杂，模板代码多：ViewModel 沦为“管道工”，充满了创建Store、转发属性和方法的样板代码。<br><br>实践中可测试性低：采用的匿名内部类实现方式，使得 Store 无法被单独实例化，单元测试变得极其困难，丧失了该模式最大的理论优势。<br><br>可读性与维护性低：业务逻辑深嵌在 ViewModel 的一个巨大内部类中，结构复杂，难以维护。 |
