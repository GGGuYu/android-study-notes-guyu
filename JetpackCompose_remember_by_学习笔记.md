# Jetpack Compose - remember 与 by 详解

## 一句话总结

- **`remember`**：在 Compose 重组时缓存值，防止状态丢失
- **`by`**：Kotlin 委托语法糖，简化 State 对象的读写（自动处理 `.value`）

---

## 1. remember 的作用

### 问题背景
Compose 是声明式 UI，当数据变化时，**整个 Composable 函数会重新执行**。

```kotlin
@Composable
fun Counter() {
    // ❌ 错误：每次重组 count 都会重置为 0
    var count = 0
    
    Button(onClick = { count++ }) {
        Text("点击了 $count 次")  // 永远显示 "0"
    }
}
```

### 解决方案
```kotlin
@Composable
fun Counter() {
    // ✅ 正确：remember 让 count 在重组时保持值
    var count by remember { mutableStateOf(0) }
    
    Button(onClick = { count++ }) {
        Text("点击了 $count 次")  // 正确显示 1, 2, 3...
    }
}
```

### 核心机制
- **第 1 次执行**：执行初始化代码，创建 State 对象，**标记并记住位置**
- **第 N 次重组**：发现该位置已"记住"，直接返回缓存的值，**跳过初始化**

---

## 2. by 的作用

### 不用 by 的麻烦写法
```kotlin
val countState = remember { mutableStateOf(0) }

// 读取值：必须加 .value
Text("${countState.value}")

// 修改值：必须加 .value
countState.value = 5
```

### 用 by 的简洁写法
```kotlin
var count by remember { mutableStateOf(0) }

// 直接像普通变量一样使用
Text("$count")
count = 5
```

### 原理解析
`by` 是 Kotlin 的**委托属性**语法，会自动：
- **读取时**：调用 `state.value`
- **写入时**：调用 `state.value = newValue`

---

## 3. 使用场景对比

| 场景 | 是否用 by | 写法示例 |
|------|----------|---------|
| **简单数据**（Int、String、Boolean） | ✅ 用 by | `var name by remember { mutableStateOf("Tom") }` |
| **复杂对象**（PagerState、LazyListState） | ❌ 不用 by | `val pagerState = rememberPagerState { 5 }` |

### 复杂对象示例
```kotlin
// ❌ 不能用 by，因为需要调用对象方法
val pagerState = rememberPagerState { datum.size }

// 通过对象访问属性和方法
val currentPage = pagerState.currentPage
pagerState.animateScrollToPage(2)
```

---

## 4. 记忆口诀

> **`remember` = 防丢失（缓存值）**  
> **`by` = 图方便（不用写 .value）**

---

## 5. 常见 API

```kotlin
// 基础状态记忆
val state = remember { mutableStateOf(initialValue) }

// 带 key 的记忆（key 变化时重新计算）
val value = remember(key) { expensiveCalculation() }

// 跨配置变化保持（如屏幕旋转）
val value = rememberSaveable { mutableStateOf(initialValue) }

// Pager 专用
val pagerState = rememberPagerState { pageCount }

// 列表滚动位置
val listState = rememberLazyListState()

// 滚动状态
val scrollState = rememberScrollState()
```

---

## 6. 重要区分

| 概念 | 归属 | 作用 |
|------|------|------|
| `remember` | **Compose 库** | 缓存计算结果，防止重组时重新创建 |
| `rememberXxx()` | **Compose 库** | 封装好的专用记忆函数（如 rememberPagerState） |
| `by` | **Kotlin 语言** | 委托属性语法，简化 getter/setter 调用 |

---

## 7. 常见错误

```kotlin
// ❌ 错误：PagerState 不能直接 mutableStateOf
var pagerState by remember { PagerState() }

// ❌ 错误：用 by 后不能调用对象方法
var count by remember { mutableStateOf(0) }
count.scrollToPage(2)  // 报错！count 是 Int

// ✅ 正确：复杂对象不用 by
val pagerState = rememberPagerState { 5 }
pagerState.scrollToPage(2)
```

---

**创建时间**: 2025-03-18  
**标签**: Jetpack Compose, Kotlin, 状态管理
