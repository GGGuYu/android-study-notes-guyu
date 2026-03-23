# Kotlin 协程学习笔记

## 目录

- [1. 快速复习卡片](#1-快速复习卡片)
- [2. 协程是什么](#2-协程是什么)
- [3. 核心概念详解](#3-核心概念详解)
- [4. Scope 生命周期管理](#4-scope-生命周期管理)
- [5. 线程调度](#5-线程调度)
- [6. 常见误区与正解](#6-常见误区与正解)
- [7. 实战示例](#7-实战示例)

---

## 1. 快速复习卡片

### 核心关键词

| 关键词 | 一句话理解 |
|--------|-----------|
| **suspend** | 标记函数可以"挂起"（暂停执行，之后恢复） |
| **CoroutineScope** | 协程的作用域，控制协程的生命周期 |
| **launch** | 启动新协程（不返回结果） |
| **async/await** | 启动协程并返回 Deferred，可获取结果 |
| **Dispatchers** | 决定协程在哪个线程执行 |
| **withContext** | 切换线程，执行完后自动返回原线程 |
| **Job** | 协程的句柄，可取消、可等待 |
| **Flow** | 响应式数据流（冷流），类似 RxJava Observable |
| **StateFlow** | 状态流（热流），有初始值，可多个观察者 |
| **stateIn** | 将 Flow 转为 StateFlow，需要 Scope |

### 核心特性

```
协程 ≈ 轻量级线程 + 可挂起恢复 + 结构化并发

轻量级：几万个协程轻松创建（线程只能几百个）
可挂起：suspend 函数可在等待时挂起，不阻塞线程
结构化：子协程随父协程取消而取消
```

### 生命周期绑定

```
GlobalScope          → 永不取消（❌ 不推荐）
viewModelScope       → ViewModel 销毁时取消
lifecycleScope       → Activity/Fragment 销毁时取消
rememberCoroutineScope() → Composable 离开组合时取消
```

---

## 2. 协程是什么

### 2.1 通俗理解

**协程 = 可以"暂停"和"继续"的函数**

想象你看电视剧按了暂停键，去做别的事，回来点播放继续看，**不会影响别人的电视机**。

```kotlin
// 普通函数：一旦开始就必须跑完
fun loadData() {
    step1()  // 开始
    step2()  // 必须等 step1 完成
    step3()  // 必须等 step2 完成
}  // 全部跑完才结束

// 协程函数：可以中途挂起，让出线程
suspend fun loadData() {
    step1()                    // 开始
    val data = fetchNetwork()  // ← 挂起点：等待网络时让出线程
    step2(data)                // 网络完成，继续执行（可能在同一线程）
    step3()                    // 继续
}
```

### 2.2 为什么会有协程

**传统回调地狱：**
```kotlin
api.getUser("123", object : Callback<User> {
    override fun onSuccess(user: User) {
        api.getOrders(user.id, object : Callback<List<Order>> {
            override fun onSuccess(orders: List<Order>) {
                api.getDetails(orders[0].id, object : Callback<Details> {
                    override fun onSuccess(details: Details) {
                        updateUI(details)
                    }
                })
            }
        })
    }
})
```

**协程写法（同步外观，异步执行）：**
```kotlin
val user = api.getUser("123")           // 异步获取
val orders = api.getOrders(user.id)     // 异步获取
val details = api.getDetails(orders[0].id) // 异步获取
updateUI(details)
```

---

## 3. 核心概念详解

### 3.1 suspend 函数的本质

**误区**：`suspend` 函数会自动切线程 ❌  
**正解**：`suspend` 只是标记可以挂起，**线程切换靠 Dispatcher**

```kotlin
// ❌ 错误：没有指定线程，会在调用线程执行（可能是主线程）
suspend fun getUser(): User {
    return api.fetchUser()  // 同步调用会阻塞当前线程！
}

// ✅ 正确：显式指定在 IO 线程执行
suspend fun getUser(): User {
    return withContext(Dispatchers.IO) {
        api.fetchUser()     // 在 IO 线程执行
    }
    // 执行完后自动回到调用线程
}
```

### 3.2 挂起的工作原理

```kotlin
suspend fun loadData() {
    println("1. 开始加载")
    val data = withContext(Dispatchers.IO) {
        println("2. 在 IO 线程执行网络请求")
        fetchNetwork()  // 模拟耗时操作
    }
    println("3. 回到主线程，数据: $data")
}

// 执行流程：
// 主线程: [1. 开始加载] → [挂起，让出主线程] 
// IO 线程:                    [2. 在 IO 线程执行网络请求]
// 主线程:                                               [3. 回到主线程]
```

**编译器转换**（简化理解）：
```kotlin
// 你写的：
suspend fun foo() {
    step1()
    step2()  // suspend 点
    step3()
}

// 编译器生成（类似状态机）：
fun foo(continuation: Continuation) {
    when (continuation.label) {
        0 -> { step1(); continuation.label = 1; step2(continuation) }
        1 -> { step3(); continuation.resume(Unit) }
    }
}
```

### 3.3 结构化并发

**父协程管理子协程**：

```kotlin
viewModelScope.launch {      // 父协程
    println("父协程开始")
    
    launch {                 // 子协程 1
        delay(1000)
        println("子协程 1 完成")
    }
    
    launch {                 // 子协程 2
        delay(2000)
        println("子协程 2 完成")
    }
    
    println("父协程结束")
} // 当 ViewModel 清除时，父 + 所有子协程一起取消
```

**错误传播**：
```kotlin
viewModelScope.launch {
    try {
        coroutineScope {     // 等待所有子协程完成
            launch {
                throw Exception("子协程出错")
            }
            launch {
                delay(1000)  // 这个也会被取消
            }
        }
    } catch (e: Exception) {
        println("捕获错误: ${e.message}")
    }
}
```

---

## 4. Scope 生命周期管理

### 4.1 各种 Scope 对比

```kotlin
// ❌ GlobalScope - 全局生命周期，永不自动取消
GlobalScope.launch {
    // 即使 Activity 销毁了还在跑，可能内存泄漏！
}

// ✅ viewModelScope - 绑定 ViewModel 生命周期
class MyViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch {
            // ViewModel 清除时自动取消
        }
    }
}

// ✅ lifecycleScope - 绑定 Activity/Fragment 生命周期
class MyActivity : AppCompatActivity() {
    fun loadData() {
        lifecycleScope.launch {
            // onDestroy() 时自动取消
        }
    }
}

// ✅ rememberCoroutineScope - 绑定 Composable 生命周期
@Composable
fun MyScreen() {
    val scope = rememberCoroutineScope()
    
    Button(onClick = {
        scope.launch {
            // Composable 离开组合时自动取消
        }
    }) { }
}
```

### 4.2 Scope 在 stateIn 中的使用

**场景**：把 Flow 转为 StateFlow，需要在后台持续收集

```kotlin
class MyAppUiState(
    coroutineScope: CoroutineScope,  // ← 传入 scope
    userDataRepository: UserDataRepository,
) {
    // Flow → StateFlow 转换需要 scope
    val userData: StateFlow<UserData> = userDataRepository.userData
        .stateIn(
            scope = coroutineScope,              // scope 启动收集
            initialValue = UserData(),           // 初始值
            started = SharingStarted.WhileSubscribed(5000) // 5秒内无人观察则停止
        )
}

// 使用
@Composable
fun rememberMyAppUiState(repository: UserDataRepository): MyAppUiState {
    val scope = rememberCoroutineScope()  // Composable 级别的 scope
    return remember(repository, scope) {
        MyAppUiState(scope, repository)
    }
}
```

**为什么需要 scope？**
- `stateIn()` 需要在后台持续收集 Flow
- 这个收集是**长期运行**的协程
- 需要绑定生命周期，防止内存泄漏

---

## 5. 线程调度

### 5.1 Dispatcher 详解

```kotlin
// Dispatchers.Main - 主线程（UI 更新）
launch(Dispatchers.Main) {
    updateUI()  // 可以操作 UI
}

// Dispatchers.IO - IO 线程（网络、文件）
launch(Dispatchers.IO) {
    file.read()     // 阻塞 IO 操作
    api.fetch()     // 网络请求
}

// Dispatchers.Default - CPU 密集型任务
launch(Dispatchers.Default) {
    list.sort()          // 大量计算
    image.process()      // 图像处理
}

// Dispatchers.Unconfined - 不指定（很少用）
launch(Dispatchers.Unconfined) {
    // 在调用线程启动，suspend 后可能在其他线程恢复
}
```

### 5.2 withContext 线程切换

```kotlin
viewModelScope.launch(Dispatchers.Main) {
    // 在主线程
    showLoading()
    
    val data = withContext(Dispatchers.IO) {
        // 切换到 IO 线程执行网络请求
        api.fetchData()
    }  // ← 自动回到主线程
    
    // 回到主线程
    updateUI(data)
    hideLoading()
}
```

### 5.3 同时执行多个任务

```kotlin
// 方式一：async/await（并发执行，等待结果）
viewModelScope.launch {
    val deferred1 = async { api.getUser() }      // 立即开始
    val deferred2 = async { api.getOrders() }    // 立即开始（并行）
    
    val user = deferred1.await()      // 等待用户数据
    val orders = deferred2.await()    // 等待订单数据（可能已经完成了）
    
    updateUI(user, orders)
}

// 方式二：async 的 awaitAll（等待所有）
viewModelScope.launch {
    val users = listOf("1", "2", "3").map { id ->
        async { api.getUser(id) }  // 同时启动多个请求
    }.awaitAll()  // 等待全部完成
}

// 方式三：coroutineScope（结构化并发）
viewModelScope.launch {
    coroutineScope {
        launch { task1() }  // 同时执行
        launch { task2() }
        launch { task3() }
    }  // 等所有子协程完成后才继续
    println("所有任务完成")
}
```

---

## 6. 常见误区与正解

### 误区 1：suspend 函数自动切线程 ❌

```kotlin
// ❌ 错误理解
suspend fun getData() {
    // 以为自动在后台线程
    return db.query()  // 实际在主线程执行！
}

// ✅ 正确
suspend fun getData() {
    return withContext(Dispatchers.IO) {
        db.query()
    }
}

// ✅ Retrofit 协程支持（自动切线程）
interface Api {
    @GET("/user")
    suspend fun getUser(): User  // Retrofit 会自动在 IO 线程执行
}
```

### 误区 2：launch 会阻塞当前线程 ❌

```kotlin
fun test() {
    println("开始")
    
    lifecycleScope.launch {
        delay(1000)  // 挂起 1 秒
        println("协程完成")
    }
    
    println("结束")  // ← 这行会立即执行，不会被阻塞
}

// 输出：
// 开始
// 结束
// （1秒后）协程完成
```

### 误区 3：try/catch 能捕获所有异常 ❌

```kotlin
viewModelScope.launch {
    try {
        api.fetchData()  // 抛出异常
    } catch (e: Exception) {
        // 能捕获到
    }
}

// 但如果不用 try/catch，异常会在 CoroutineExceptionHandler 处理
// 或导致协程取消
```

### 误区 4：取消协程会立即停止 ❌

```kotlin
val job = viewModelScope.launch {
    try {
        while (true) {
            doWork()
            delay(1000)  // ← 取消点：可以被取消
        }
    } catch (e: CancellationException) {
        // 清理资源
        cleanup()
        throw e  // 必须重新抛出！
    }
}

job.cancel()  // 协程会在下一个 "挂起点" 取消
```

**协作式取消**：
```kotlin
// 检查是否被取消
while (isActive) {  // 手动检查
    doWork()
}

// 或确保有挂起点
while (true) {
    doWork()
    yield()  // 检查取消并让步
}
```

---

## 7. 实战示例

### 7.1 ViewModel + Repository 模式

```kotlin
// Repository
class UserRepository @Inject constructor(
    private val api: UserApi,
    private val db: UserDao
) {
    suspend fun getUser(id: String): User {
        return withContext(Dispatchers.IO) {
            // 先查本地缓存
            val local = db.getUser(id)
            if (local != null) return@withContext local
            
            // 再请求网络
            val remote = api.getUser(id)
            db.saveUser(remote)  // 保存到本地
            remote
        }
    }
}

// ViewModel
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    
    private val _user = MutableStateFlow<User?>(null)
    val user: StateFlow<User?> = _user.asStateFlow()
    
    fun loadUser(id: String) {
        viewModelScope.launch {
            try {
                _user.value = repository.getUser(id)
            } catch (e: Exception) {
                // 处理错误
            }
        }
    }
}

// Compose
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val user by viewModel.user.collectAsStateWithLifecycle()
    
    // UI 自动更新
}
```

### 7.2 Flow + stateIn 实现数据共享

```kotlin
class UserRepository @Inject constructor(
    private val dataStore: DataStore<UserPreferences>
) {
    // 冷流：每个收集者独立
    val userData: Flow<UserData> = dataStore.data
        .map { it.toUserData() }
    
    // 需要在 ViewModel 转成 StateFlow
}

class UserViewModel @Inject constructor(
    repository: UserRepository
) : ViewModel() {
    
    // 转成 StateFlow，多个 UI 组件共享
    val userData: StateFlow<UserData> = repository.userData
        .stateIn(
            scope = viewModelScope,
            initialValue = UserData(),
            started = SharingStarted.WhileSubscribed(5000)
        )
}
```

### 7.3 带超时和重试的请求

```kotlin
viewModelScope.launch {
    try {
        val data = withTimeout(5000) {  // 5秒超时
            retry(times = 3) {           // 重试3次
                api.fetchData()
            }
        }
        updateUI(data)
    } catch (e: TimeoutCancellationException) {
        showError("请求超时")
    }
}
```

---

## 总结

### 核心口诀

> **suspend 标记挂起点，scope 绑定生命周期**  
> **launch 启动不返回，async 返回可等待**  
> **withContext 切线程，flow 数据可观察**  
> **stateIn 转热流，多个 UI 共享数据**

### 学习路径建议

```
1. 理解 suspend 和挂起恢复
   ↓
2. 掌握 launch/async/await
   ↓
3. 学会使用各种 Scope
   ↓
4. 理解 Dispatcher 线程切换
   ↓
5. 掌握 Flow 和 StateFlow
   ↓
6. 实战项目练习
```

---

**创建时间**: 2025-03-19  
**标签**: Kotlin, Coroutine, Android, 异步编程, Flow