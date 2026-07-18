---
title: 从 Hello Android 到本地记账 App：Room、StateFlow 与 Compose 的完整实践
excerpt: 复盘一款本地记账 App 的数据层、响应式 UI、数据库迁移与依赖兼容问题。
date: 2026-07-18 11:22:21 +0800
categories:
  - Android
  - 移动开发
tags:
  - Kotlin
  - Jetpack Compose
  - Room
  - StateFlow
  - KSP
---

这次项目从 Android Studio 新建工程自带的 `Hello Android` 页面开始，目标是做一款完全本地运行的记账 App：不依赖后端，交易记录保存在设备中，界面能够实时展示收入、支出和分类统计，同时支持新增、编辑和删除流水。

功能本身并不复杂，但真正把一个模板工程推进到可用状态，还是会遇到数据建模、响应式状态、数据库升级和构建工具兼容等问题。本文记录这次实现中最值得复用的设计和排查过程。

## 技术选型：保持本地优先和单向数据流

项目采用 Kotlin 和 Jetpack Compose 编写界面，Room 负责本地持久化，ViewModel、协程和 StateFlow 负责连接数据库与 UI。当前工程的主要版本如下：

| 组件 | 版本 |
| --- | --- |
| Android Gradle Plugin | 9.3.0 |
| Kotlin | 2.2.10 |
| Jetpack Compose BOM | 2026.02.01 |
| Room | 2.8.4 |
| KSP | 2.3.10 |

选择 Room 而不是直接操作 SQLite，主要是为了获得编译期 SQL 校验、类型安全的 DAO 和 Flow 查询。Compose 侧不主动刷新数据，而是订阅 ViewModel 暴露的状态：数据库发生变化后，Room 发出新列表，StateFlow 更新，界面自动重组。

整条数据链路可以概括为：

```text
Compose UI -> ViewModel -> TransactionDao -> Room/SQLite
     ^                                      |
     +------------- StateFlow <--- Flow ----+
```

## Room 数据层：一条流水需要哪些信息

最初的交易模型只有金额、收支类型、备注和时间。后来加入分类能力后，实体扩展为六个业务字段：

```kotlin
@Entity(tableName = "transactions")
data class Transaction(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0L,
    val amount: Double,
    val type: Int,
    val category: String,
    val note: String,
    val timestamp: Long
)
```

`type` 使用 `0` 表示支出、`1` 表示收入。金额始终保存为正数，正负号只在展示层根据类型决定。这样做能让收入和支出的聚合查询更直接，也避免同一字段同时承担“数值”和“业务类型”两种含义。

DAO 除了基本的增删改，还直接暴露按时间倒序的 Flow，并在 SQL 层完成总额统计：

```kotlin
@Query("SELECT * FROM transactions ORDER BY timestamp DESC")
fun getAllTransactions(): Flow<List<Transaction>>

@Query("SELECT COALESCE(SUM(amount), 0.0) FROM transactions WHERE type = 1")
fun getTotalIncome(): Flow<Double>

@Query("SELECT COALESCE(SUM(amount), 0.0) FROM transactions WHERE type = 0")
fun getTotalExpense(): Flow<Double>
```

这里的 `COALESCE` 很重要。空表执行 `SUM` 会得到 `NULL`，但 UI 需要稳定的 `Double`。在查询层将空结果转换为 `0.0`，可以避免把可空类型和默认值判断扩散到 ViewModel 与 Compose。

## ViewModel：把冷 Flow 转成 UI 状态

Room 返回的是 Flow，而 Compose 页面更适合消费具有当前值的 StateFlow。ViewModel 使用 `stateIn` 完成转换：

```kotlin
val transactions: StateFlow<List<Transaction>> = transactionDao
    .getAllTransactions()
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = emptyList()
    )
```

`SharingStarted.WhileSubscribed(5_000)` 表示页面没有订阅者后保留上游五秒。短暂的配置变化不会立刻停止数据流，长期离开页面又不会一直占用资源。新增、编辑和删除则在 `viewModelScope` 中调用 DAO，UI 不直接接触数据库实例。

Compose 侧通过 `collectAsStateWithLifecycle()` 收集状态，因此页面进入非活跃生命周期后不会继续无意义地刷新。这条边界让 Activity 只负责创建 ViewModel 和设置主题，具体账本逻辑集中在 `LedgerScreen` 中。

## UI 演进：从记录列表到月度账本

第一版页面只有两张统计卡、一列流水和一个新增按钮。它能工作，但不符合实际记账习惯：用户通常关心“这个月花了多少”“钱花在哪些分类”“某一天有哪些流水”，而不是全生命周期总额。

新版页面以月份为主视角。用户可以切换月份，页面从完整交易列表中筛出当月记录，再计算收入、支出和结余：

```kotlin
val monthTransactions = remember(transactions, selectedMonth) {
    transactions.filter { monthOf(it.timestamp) == selectedMonth }
}

val monthlyIncome = remember(monthTransactions) {
    monthTransactions.filter { it.type == 1 }.sumOf { it.amount }
}

val monthlyExpense = remember(monthTransactions) {
    monthTransactions.filter { it.type == 0 }.sumOf { it.amount }
}
```

流水页面增加了“全部、支出、收入”筛选，并按 `LocalDate` 分组。每个日期标题同时展示当天收入与支出，单条记录展示分类、备注、记账时间和带颜色的金额。分类分析页再按类别汇总当月支出，用进度条展示金额和占比。

新增与编辑共用同一个 Material 3 Bottom Sheet。表单支持金额、收支类型、预设或自定义分类、日期、记账时间和备注。点击已有流水会带入原值，保存时执行 `@Update`；删除操作会先弹出二次确认，避免误触。

视觉上没有照搬某款产品，而是采用适合工具类应用的数据优先布局：墨绿色用于月度总览，绿色表示收入，珊瑚红表示支出。设备截图检查时还发现，浮动新增按钮会遮住分类分析的最后一项，因此最终只在流水页显示 FAB。这个问题编译器无法发现，必须在真实尺寸上检查。

## 数据库升级：增加分类不能直接改实体

分类字段是在数据库已经存在之后加入的。如果只修改 Entity 并把版本号从 1 改为 2，旧用户的数据库结构与 Room 期望结构不一致，启动时就会失败。

项目使用显式迁移为旧表增加非空字段，并给历史流水设置“其他”作为默认分类：

```kotlin
private val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL(
            "ALTER TABLE transactions ADD COLUMN category TEXT NOT NULL DEFAULT '其他'"
        )
    }
}
```

迁移通过 `Room.databaseBuilder(...).addMigrations(MIGRATION_1_2)` 注册。相比 `fallbackToDestructiveMigration()`，这种做法不会为了省事清空用户账本。记账数据属于用户资产，即使应用仍处于早期阶段，也应该从第一次结构变化开始认真维护迁移路径。

## 踩坑一：依赖要求 API 37，本机只有 Android 36.1

第一次加入 Lifecycle Compose 依赖后，`checkDebugAarMetadata` 报告三个错误：`core-ktx:1.19.0`、`core:1.19.0` 和 `lifecycle-runtime-compose-android:2.11.0` 都要求 `compileSdk` 至少为 37，而项目使用的是 Android 36.1。

这里容易被错误信息末尾的 `minCompileSdk` 误导。修改应用的 `minSdk` 没有作用，因为问题是编译时 API 版本不够，不是设备最低安装版本。

本机当时只安装了 `android-36.1`，因此我没有直接把 `compileSdk` 改到一个不存在的 37.1，而是将依赖锁定到兼容版本：

```toml
coreKtx = "1.18.0"
lifecycleRuntimeKtx = "2.10.0"
```

随后单独运行 `:app:checkDebugAarMetadata`，任务通过。这个处理并不是说依赖永远不升级，而是让项目配置与本机 SDK 保持一致；等安装 Android 37 后，再整体升级和验证更合理。

## 踩坑二：AGP 9 内置 Kotlin 与旧 KSP 冲突

Room 编译器最初接入 KSP `2.2.10-2.0.2` 后，Gradle 在配置阶段失败，核心提示是：

```text
Using kotlin.sourceSets DSL to add Kotlin sources is not allowed
with built-in Kotlin.
```

AGP 9 使用内置 Kotlin，旧版 KSP 仍通过 `kotlin.sourceSets` 注册生成目录，两者发生冲突。错误提示提供了放宽检查的兼容开关，但那只是绕过限制，没有解决插件集成本身已经过时的问题。

最终将 KSP 升级到独立版本 `2.3.10`，重新执行 `:app:assembleDebug` 后，`kspDebugKotlin`、`compileDebugKotlin` 和 APK 打包全部通过。这个问题说明：KSP 版本不能只机械地跟随 Kotlin 版本字符串，还要考虑当前 AGP 的 Kotlin 集成方式。

## 结果与下一步

目前应用已经具备一款本地记账工具的核心闭环：

- Room 持久化交易数据；
- 收入、支出、分类、备注和自定义时间；
- 月份切换、月度结余和按日流水；
- 收支筛选与分类支出分析；
- 新增、编辑和带确认的删除；
- v1 到 v2 的无损数据库迁移。

项目已经通过 `assembleDebug` 构建，并在 Android 模拟器上检查了首页、分析页和编辑表单。但它还不是完整的商业记账产品：目前没有预算提醒、搜索、账单导出、云同步和自动化测试，这些能力不应该只靠继续堆 UI 来实现。

下一步我会优先补两件事。第一是把月度筛选和分类聚合下沉到 DAO，避免数据量增大后在 Compose 层遍历全部记录；第二是为迁移、金额统计和 ViewModel 操作补充自动化测试。等数据正确性有保障后，再考虑预算和导出功能。

从模板页到可用 App，真正有价值的并不是写出了多少个 Composable，而是把数据库、状态流、界面和构建工具之间的边界理顺。对于本地优先应用，这些基础决定了后续功能是稳定演进，还是每次升级都要担心用户数据和构建环境。
