---
title: "现代化构建标准：Gradle Version Catalog 技术白皮书"
pubDatetime: 2025-11-22T23:05:35+08:00
slug: gradle-version-catalog
tags: ["Android", "gradle"]
description: "Gradle Version Catalog（版本目录） 是 Gradle 官方（7.0+）推出的一种集中式依赖管理标准。"
---


## 1. 什么是 Gradle Version Catalog？

**Gradle Version Catalog（版本目录）** 是 Gradle 官方（7.0+）推出的一种**集中式依赖管理标准**。

它允许开发者在一个独立的配置文件（标准命名为 `libs.versions.toml`）中，声明项目中所有的：

  * **依赖库 (Libraries)**：如 Retrofit, Room, Compose 等。
  * **版本号 (Versions)**：统一管理的版本常量。
  * **插件 (Plugins)**：如 Android Gradle Plugin, Kotlin Plugin 等。
  * **依赖组合 (Bundles)**：将多个相关的库打包成一个集合。

Gradle 会在构建过程中解析该文件，并自动生成 **类型安全（Type-Safe）** 的访问器类，供各个模块的构建脚本直接调用。

> **核心理念**：将“依赖的配置（定义）”与“依赖的使用（声明）”彻底解耦，以此替代传统的 `ext` 扩展属性或 `buildSrc` 方案。

## 2. 为什么要升级？(VS config.gradle)

目前我们项目使用的 `config.gradle` (Ext 属性) 方案虽然解决了“版本统一”的问题，但在开发体验和工程化质量上存在明显瓶颈。以下是深度对比：

| 核心维度 | 旧方案 (config.gradle) | 新方案 (Version Catalog) |
| :--- | :--- | :--- |
| **IDE 支持** | **无**。完全靠记忆 key 或复制粘贴，容易拼写错误。 | **完美**。Android Studio 提供自动补全、代码高亮和重构支持。 |
| **导航体验** | **弱**。无法直观查看库的详细坐标。 | **强**。`Ctrl+Click` 直接跳转到 TOML 定义处，一目了然。 |
| **类型安全** | **弱**。依赖引用本质是字符串，写错要等到 Sync 或运行才报错。 | **强**。编译时生成代码，写错直接标红（Compile-time check）。 |
| **依赖成组** | **繁琐**。需手动定义 List 并编写 groovy 闭包进行遍历。 | **原生支持** (Bundles)。一行代码引入全套“全家桶”依赖。 |
| **DSL 兼容性** | **差**。在 `.gradle.kts` 中调用 `ext` 属性语法非常丑陋。 | **优**。原生支持 Kotlin DSL，语法流畅自然。 |
| **版本检查** | **无**。无法感知是否有新版本。 | **有**。IDE 会在 TOML 文件中高亮提示过期库，支持一键升级。 |

## 3. 核心配置详解：`libs.versions.toml`

标准路径：项目根目录下的 `gradle/libs.versions.toml`。该文件由四个核心板块组成。

### 3.1 `[versions]` —— 版本号仓库

定义项目中用到的所有版本号变量。建议使用 `camelCase` (驼峰) 命名。

```toml file="libs.version.toml"
[versions]
# Project Specs
androidMinSdk = "24"
androidTargetSdk = "34"

# Core Dependencies
kotlin = "1.9.22"
coroutines = "1.7.3"
retrofit = "2.9.0"
okhttp = "4.12.0"
room = "2.6.1"
```

### 3.2 `[libraries]` —— 依赖库定义

这里是依赖管理的核心。

  * **语法映射**：TOML 中的短横线 `-` 在代码中会被映射为点号 `.`。
  * **别名规范**：建议使用层级命名法（如 `androidx-core-ktx`），生成的代码为 `libs.androidx.core.ktx`。

```toml file="libs.version.toml"
[libraries]
# 方式 1: 使用 module 简写 (推荐，简洁)
retrofit-core = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
retrofit-gson = { module = "com.squareup.retrofit2:converter-gson", version.ref = "retrofit" }
okhttp-logging = { module = "com.squareup.okhttp3:logging-interceptor", version.ref = "okhttp" }

# 方式 2: group 和 name 分离 (清晰)
androidx-core-ktx = { group = "androidx.core", name = "core-ktx", version.ref = "androidxCore" }

# 方式 3: 使用 BOM (Bill of Materials)
# 定义 BOM
compose-bom = { group = "androidx.compose", name = "compose-bom", version = "2025.11.01" }
# 使用 BOM 内的库 (不需要 version)
compose-ui = { group = "androidx.compose.ui", name = "ui" }
```

### 3.3 `[bundles]` —— 依赖打包

这是 Version Catalog 相比 `config.gradle` 最大的优势之一。我们可以将一组逻辑强相关的库打包。

**应用场景**：网络层全家桶、Compose UI 套件、测试库集合等。

```toml file="libs.version.toml"
[bundles]
# 定义：网络请求套件 (Retrofit + Gson + OkHttp Log)
networking = ["retrofit-core", "retrofit-gson", "okhttp-logging"]

# 定义：Room 数据库套件
database = ["room-runtime", "room-ktx"]
```

### 3.4 `[plugins]` —— 插件管理

统一管理插件 ID 和版本，解决多模块间插件版本冲突问题。

```toml file="libs.version.toml"
[plugins]
android-application = { id = "com.android.application", version = "8.2.0" }
android-library = { id = "com.android.library", version = "8.2.0" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
ksp = { id = "com.google.devtools.ksp", version = "1.9.22-1.0.17" }
```

## 4. 实战：如何在代码中使用

Gradle 会根据 TOML 配置自动生成名为 `libs` 的全局访问器。

### 4.1 引入依赖 (Dependencies)

**旧方式 (config.gradle):**

```groovy file="build.gradle"
// 繁琐，无补全
implementation rootProject.ext.deps.retrofit
implementation rootProject.ext.deps.gson
rootProject.ext.networkLibs.each { implementation it }
```

**新方式 (Version Catalog):**

```kotlin file="build.gradle.kts"
dependencies {
    // 1. 引入单个库 (自动补全，所见即所得)
    implementation(libs.androidx.core.ktx)
    
    // 2. 引入 Bundles (一行顶多行，极度舒适)
    implementation(libs.bundles.networking)
    implementation(libs.bundles.database)
    
    // 3. 使用 BOM 平台
    implementation(platform(libs.compose.bom))
    implementation(libs.compose.ui)
    
    // 4. 注解处理器 (KSP)
    ksp(libs.room.compiler)
}
```

### 4.2 应用插件 (Plugins)

在根目录或模块的 `build.gradle.kts` 顶部使用 `alias` 关键字。注意：`alias` 强制应用 TOML 中指定的版本，确保一致性。

```kotlin file="build.gradle.kts"
plugins {
    // 映射自 [plugins] 区域
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.ksp)
}
```

## 5. 最佳实践 Tips

* **命名规范**：
    * **Versions**: 驼峰命名 (e.g., `minSdk`, `coroutines`).
    * **Libraries**: 短横线分隔 (e.g., `androidx-lifecycle-runtime`). 这样生成的 Kotlin 代码 `libs.androidx.lifecycle.runtime` 最符合直觉。
* **保持整洁**：Android Studio 会自动检查并提示 TOML 件中的版本更新。
* **团队协作**：TOML 文件冲突通常比 `build.gradle` 冲突更容易解决，因为它纯粹是 Key-Value 对，不包逻辑。

## 6. 进阶理念：Version Catalog 与 Convention Plugins 的协同效应

下图展示了 Version Catalog、Convention Plugins 与业务模块之间的理想协作关系：

![协作关系图](/images/gradle-version-catalog.svg)

引入 Version Catalog 后，我们解决了“版本和依赖来源”的统一问题，但一个新的问题浮出水面：**配置逻辑的重复**。

每个模块（尤其是 `feature` 模块）的 `build.gradle.kts` 文件仍然需要重复编写大量样板代码，例如：
- `android { ... }` 配置块（`compileSdk`, `defaultConfig`, `buildTypes` 等）。
- 基础依赖库的声明（`core-ktx`, `appcompat`, `material` 等）。
- `testOptions` 配置。

这违反了 **DRY (Don't Repeat Yourself)** 原则。

### 6.1 为什么不使用脚本动态“生成”build.gradle文件？

在 Convention Plugins 出现之前，一些项目为了解决配置复用问题，采取了一种激进的元编程方案：**编写一个主脚本（通常是 Groovy），用它来动态生成每个模块所需要的 `build.gradle` 文件内容**。

这种方案虽然在表面上消除了重复，但引入了更深层次的工程化问题，是现代 Gradle 开发中应当极力避免的反模式：

- **事实来源（Source of Truth）混乱**：开发者在 IDE 中看到的 `build.gradle` 文件只是一个被生成的“产物”，并非真正的配置源头。真正的逻辑隐藏在那个生成脚本中，这会造成极大的认知混乱和误导。
- **构建逻辑黑盒化**：想要理解一个模块究竟是如何配置的，你无法直接查看它自己的构建脚本，而是要去阅读和理解那个复杂的生成脚本，这让构建系统变得难以捉摸。
- **IDE 支持完全失效**：IDE 的代码补全、导航和语法检查功能都是针对当前 `build.gradle` 文件的。对于背后那个动态生成逻辑的脚本，IDE 束手无策，开发者回到了“记事本编程”时代。
- **重构与调试的噩梦**：任何微小的构建调整都需要经历“修改生成脚本 -> 重新执行生成 -> 同步 Gradle”这一漫长、易错且难以调试的循环。
- **严重破坏 Gradle 缓存**：这种模式是 Gradle 性能优化的天敌。它直接导致两大缓存机制失效：
    - **配置缓存 (Configuration Cache)**：由于构建脚本被外部修改，Gradle 别无选择，只能让配置缓存失效并重新执行所有配置，使得该功能形同虚设。
    - **构建缓存 (Build Cache)**：生成脚本是一个隐式的、无法被 Gradle 稳定追踪的输入。它的存在导致任务的输入极不稳定，缓存命中率大幅降低，最终拖慢整体构建速度。

相比之下，Convention Plugins 提供了一种声明式、类型安全且对 IDE 极其友好的方式来封装和复用构建逻辑，是解决配置重复问题的正确演进方向。

### 6.2 最终答案：Convention Plugins (约定插件)

现代 Gradle 项目的最佳实践是使用 **Convention Plugins**，它与 Version Catalog 相辅相成，构成完美的构建系统。

- **Convention Plugins**：存放在 `build-logic` 目录下的 **预编译脚本插件**。它们是真正的、用 Kotlin 编写的、类型安全的插件。
- **核心思想**：将“通用构建配置”封装成一个个带有明确功能的插件。例如，我们可以创建 `com.example.android.library` 插件来为所有 Android Library 模块提供标准配置。

**两者的关系:**
1. **Version Catalog (`libs.versions.toml`)**：作为“单一数据源”，负责**定义**所有依赖和版本。
2. **Convention Plugins (`build-logic`)**：作为“配置中心”，负责**读取** Version Catalog 中的依赖，并将其与通用配置逻辑（如 `compileSdk`）**应用**到目标模块。

约定插件主要有两种实现方式：
- **预编译脚本 (`.gradle.kts`)**：直接在 `build-logic/src/main/kotlin` 目录下创建 `my-plugin.gradle.kts` 文件。这种方式简单直接，适合小型项目或简单的逻辑封装。
- **实现 `Plugin<Project>` 接口的 Kotlin 类 (`.kt`)**：在 `build-logic` 中创建标准的 Kotlin 类文件。这是 `nowinandroid` 等大型项目采用的最佳实践，它提供了更好的结构、可测试性和可扩展性。

### 6.3 实战示例 (Plugin<Project> 模式)

下面我们以 google 官方 Android 架构演示项目：`nowinandroid` 的方式，演示如何创建一个用于配置 Android Library 模块的约定插件。

**第 1 步：在 `build-logic` 中注册你的插件**

首先，在 `build-logic` 模块自己的 `build.gradle.kts` 文件中，使用 `gradlePlugin` 块来注册插件，并为其分配一个全局唯一的 ID。

```kotlin file="build.gradle.kts"
// build-logic/build.gradle.kts
import org.gradle.kotlin.dsl.gradleKotlinDsl

plugins {
    `kotlin-dsl`
}

// ...

gradlePlugin {
    plugins {
        register("androidLibrary") {
            id = "com.example.android.library" // 插件的唯一 ID
            implementationClass = "com.example.convention.AndroidLibraryConventionPlugin" // 实现该插件的类的完全限定名
        }
        // 可以注册更多其他插件，如 androidApplication, androidFeature 等
    }
}
```

**第 2 步：编写插件实现类**

接下来，创建上面 `implementationClass` 指定的 Kotlin 类。

```kotlin file="AndroidLibraryConventionPlugin.kt"
// build-logic/src/main/kotlin/com/example/convention/AndroidLibraryConventionPlugin.kt
import com.android.build.gradle.LibraryExtension
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.api.artifacts.VersionCatalogsExtension
import org.gradle.kotlin.dsl.configure
import org.gradle.kotlin.dsl.dependencies
import org.gradle.kotlin.dsl.getByType

class AndroidLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            // 获取 Version Catalog
            val libs = extensions.getByType<VersionCatalogsExtension>().named("libs")

            // 1. 应用核心插件
            pluginManager.apply("com.android.library")
            pluginManager.apply("org.jetbrains.kotlin.android")

            // 2. 配置 Android Library Extension
            extensions.configure<LibraryExtension> {
                compileSdk = libs.findVersion("androidTargetSdk").get().toString().toInt()
                
                defaultConfig {
                    minSdk = libs.findVersion("androidMinSdk").get().toString().toInt()
                    testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
                }

                compileOptions {
                    sourceCompatibility = JavaVersion.VERSION_17
                    targetCompatibility = JavaVersion.VERSION_17
                }
                // 其他通用配置...
            }

            // 3. 配置通用依赖
            dependencies {
                "implementation"(libs.findLibrary("androidx-core-ktx").get())
                "implementation"(libs.findLibrary("androidx-appcompat").get())
                "implementation"(libs.findLibrary("material").get())

                "testImplementation"(libs.findLibrary("junit").get())
                "androidTestImplementation"(libs.findLibrary("androidx-junit").get())
            }
        }
    }
}
```

**第 3 步：在业务模块中应用插件**

现在，`feature` 模块的构建脚本变得异常简洁。它只需声明它“是”一个我们定义好的 Android Library，即可获得所有通用配置。

```kotlin file="build.gradle.kts"
// feature/login/build.gradle.kts

plugins {
    // 只需通过 ID 应用我们自己的约定插件
    id("com.example.android.library")
    // 如果需要，可以再应用 Hilt 等特定插件
    alias(libs.plugins.hilt.android)
}

// 这里不再需要 android { ... } 配置块

dependencies {
    // 只需声明该模块“特有”的依赖
    implementation(libs.androidx.lifecycle.viewmodel.ktx)
    implementation(libs.bundles.networking)
    
    // Hilt KSP
    ksp(libs.hilt.android.compiler)
}
```

通过这种方式，我们将复杂的构建逻辑封装成了可测试、可维护的独立单元，极大地提升了项目的工程质量。

### 6.4 现实考量：现阶段是否需要 Convention Plugins？

**需要明确的是，Convention Plugins 并非银弹。** 它在提供极致复用和抽象的同时，也引入了额外的 `build-logic` 模块，增加了项目的复杂度和工程师的理解成本。

对于我们当前的项目而言，模块数量适中，且各个模块的 `build.gradle.kts` 变更频率并不高。在这种情况下，强行引入 Convention Plugins 可能是一种“过度设计”。

因此，在现阶段，我们做出一个务实的架构决策：

**可以接受在每个模块中存在重复的配置块（如 `android { ... }`），但必须坚守一条核心原则：所有关键的版本号、SDK 版本和依赖库，都必须从 `libs.versions.toml` 中通过类型安全的访问器获取。**

例如，一个模块的配置可以是这样的：

```kotlin file="build.gradle.kts"
// feature/another-feature/build.gradle.kts

plugins {
    alias(libs.plugins.android.library)
    alias(libs.plugins.kotlin.android)
}

// 配置块可以重复，但值必须来自 Version Catalog
android {
    compileSdk = libs.versions.android.targetSdk.get().toInt()
    defaultConfig {
        minSdk = libs.versions.android.minSdk.get().toInt()
    }
    // ...
}

dependencies {
    // 依赖同样来自 Version Catalog
    implementation(libs.androidx.core.ktx)
    implementation(libs.bundles.networking)
}
```

这种“半自动化”的方式，是在追求“完美抽象”与“当前可维护性”之间取得的平衡。它保证了整个项目在核心依赖上的一致性，同时避免了过早引入额外复杂性。当未来项目规模扩大，配置变更日益频繁时，我们再平滑地演进到完整的 Convention Plugins 方案。

## 7. 迁移指南：从 config.gradle 平滑过渡到 Version Catalog

对于仍在使用 `config.gradle` 脚本通过 `ext` 属性管理依赖的旧项目，可以遵循以下步骤无痛迁移到 Version Catalog。

### 步骤 1：创建 `gradle/libs.versions.toml` 文件

在项目根目录下的 `gradle` 文件夹中创建 `libs.versions.toml` 文件。如果 `gradle` 文件夹不存在，请先创建它。现代 Gradle 会自动识别这个约定位置的文件。

### 步骤 2：迁移版本号

将 `config.gradle` 文件中的 `versions` map 迁移到 `[versions]` 部分。

**Before (`config.gradle`):**
```groovy file="config.gradle"
ext {
    versions = [
            kotlin: "1.9.22",
            retrofit: "2.9.0"
    ]
    // ...
}
```

**After (`libs.versions.toml`):**
```toml file="libs.version.toml"
[versions]
kotlin = "1.9.22"
retrofit = "2.9.0"
```

### 步骤 3：迁移依赖库

将 `deps` map 迁移到 `[libraries]` 部分。请注意使用 `version.ref` 来引用 `[versions]` 中定义的版本。

**Before (`config.gradle`):**
```groovy file="config.gradle"
ext {
    // ...
    libdependencies = [
            "room-runtime"               : "androidx.room:room-runtime:2.4.3",
            "room-compiler"              : "androidx.room:room-compiler:2.4.3",
            "room-ktx"                   : "androidx.room:room-ktx:2.4.3",
    ]
}
```

**After (`libs.versions.toml`):**
```toml file="libs.version.toml"
[versions]
room = "2.4.3"

# ...

[libraries]
# 建议使用短横线分隔命名
androidx-room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
androidx-room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }
```

### 步骤 4：(推荐) 创建 Bundles

对于逻辑上相关的依赖组，迁移到 `[bundles]` 可以极大地简化后续使用。

**Before (`config.gradle`):**
```groovy file="config.gradle"
ext {
    // ...
    deps = [
            // ...
            "test-junit": "junit:junit:4.13.2",
            "test-androidx-junit": "androidx.test.ext:junit:1.1.5"
    ]
    testLibs = [
            deps."test-junit",
            deps."test-androidx-junit"
    ]
}
```

**After (`libs.versions.toml`):**
```toml file="libs.version.toml"
[versions]
junit = "4.13.2"
androidxJunit = "1.1.5"
# ...

[libraries]
test-junit = { module = "junit:junit", version.ref = "junit" }
test-androidx-junit = { module = "androidx.test.ext:junit", version.ref = "androidxJunit" }
# ...

[bundles]
testing = ["test-junit", "test-androidx-junit"]
```

### 步骤 5：更新模块构建脚本

这是最核心的一步。需要逐个修改模块的 `build.gradle` 文件，用类型安全的访问器替换旧的字符串引用。

**Before (`build.gradle`):**
```groovy file="build.gradle"
// 旧的 groovy ext 访问方式在 groovy 中非常丑陋
def config = rootProject.extensions.getByName("ext")

dependencies {
    implementation config.libdependencies["retrofit-core"]
    testImplementation config.libdependencies["test-junit"]
}
```

**After (`build.gradle.kts`):**
```kotlin file="build.gradle.kts"
dependencies {
    // 自动补全，类型安全，代码美观
    implementation(libs.retrofit.core)
    
    // 通过 bundle 引入一组测试依赖
    testImplementation(libs.bundles.testing)
}
```
对于**插件**的替换：

**Before (`build.gradle`):**
```groovy file="build.gradle"
plugins {
    id 'com.android.library'
}
```

**After (`libs.versions.toml` & `build.gradle.kts`):**
```toml file="libs.version.toml"
# In libs.versions.toml
[plugins]
android-library = { id = "com.android.library", version = "8.2.0" }
```

```kotlin file="build.gradle.kts"
# In build.gradle.kts
plugins {
    alias(libs.plugins.android.library)
}
```

### 步骤 6：清理旧配置

当确认所有模块都已成功迁移并能正常编译后，就可以安全地：
1.  删除项目根目录下的 `config.gradle` 文件。
2.  删除根 `build.gradle` 文件中 `def config = rootProject.extensions.getByName("ext")` 这一行。

### 迁移注意事项

- **命名转换规则**：这是最关键的一点。TOML 文件中的别名会转换为 Kotlin DSL 中的访问器。转换规则通常是：`kebab-case` (短横线) 和 `snake_case` (下划线) 都会被转换为 `camelCase` 或 `dot.case`。例如，`retrofit-core` 变为 `libs.retrofit.core`。请在 IDE 中尝试输入 `libs.`，利用自动补全来探索和确认正确的访问器名称。
- **清理和同步**：迁移完成后，强烈建议执行 `./gradlew clean` 命令，并在 Android Studio 中点击 `Sync Project with Gradle Files`。这会强制 Gradle 清理旧的缓存并生成最新的类型安全访问器。
- **渐进式迁移**：你不必一次性迁移所有模块。Version Catalog 可以和旧的 `ext` 属性共存。你可以先创建一个 `libs.versions.toml` 文件，然后逐个模块进行改造，验证通过后再继续下一个，这是一种更安全、风险更低的迁移策略。
- **插件版本**：在 `[plugins]` 中定义的插件，使用 `alias(libs.plugins.some.plugin)` 会强制使用 TOML 中定义的版本，有助于统一插件版本，但这也意味着 `plugins` 块中的 `version "..."` 写法会与之冲突。
