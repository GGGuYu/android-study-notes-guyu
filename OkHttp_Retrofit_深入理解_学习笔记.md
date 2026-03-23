# OkHttp + Retrofit 深入理解笔记

## 目录

- [1. 快速复习卡片](#1-快速复习卡片)
- [2. OkHttp 核心架构](#2-okhttp-核心架构)
  - [2.1 拦截链（Interceptors）](#21-拦截链interceptors)
  - [2.2 分发器（Dispatcher）](#22-分发器dispatcher)
  - [2.3 连接池（ConnectionPool）](#23-连接池connectionpool)
- [3. Retrofit 核心机制](#3-retrofit-核心机制)
  - [3.1 动态代理原理](#31-动态代理原理)
  - [3.2 Converter 数据转换](#32-converter-数据转换)
  - [3.3 CallAdapter 调用适配](#33-calladapter-调用适配)
- [4. 完整请求流程图解](#4-完整请求流程图解)
- [5. 项目实战配置分析](#5-项目实战配置分析)
- [6. 性能优化与面试话术](#6-性能优化与面试话术)
- [7. 常见误区与正解](#7-常见误区与正解)

---

## 1. 快速复习卡片

### 核心关键词

| 关键词 | 一句话理解 |
|--------|-----------|
| **Interceptor** | 拦截器，处理日志、缓存、认证、重试等 |
| **Dispatcher** | 分发器，管理并发请求和线程调度 |
| **ConnectionPool** | 连接池，复用 TCP 连接避免重复握手 |
| **Converter** | 数据转换器，JSON ↔ Kotlin 对象 |
| **CallAdapter** | 调用适配器，适配不同返回值类型（suspend/RxJava）|
| **ServiceMethod** | Retrofit 缓存的方法解析结果 |
| **动态代理** | 运行时生成接口实现，拦截方法调用 |
| **keep-alive** | HTTP/1.1 连接复用机制 |
| **HTTP/2 多路复用** | 一个 TCP 连接并行传输多个请求 |

### OkHttp vs Retrofit 职责划分

```
┌─────────────────────────────────────┐
│           Retrofit 层                │
│  - 注解解析（@GET/@POST/@Path）       │
│  - 动态代理生成接口实现               │
│  - Converter（JSON ↔ 对象转换）       │
│  - ServiceMethod 缓存                │
└─────────────┬───────────────────────┘
              ↓ 构建 Request
┌─────────────────────────────────────┐
│           OkHttp 层                  │
│  ┌───────────────────────────────┐  │
│  │     拦截链（Interceptors）     │  │
│  │  应用拦截器 → 重试 → 桥接 →   │  │
│  │  缓存 → 连接 → 网络拦截器      │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │     分发器（Dispatcher）       │  │
│  │  管理并发请求队列和线程池       │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │     连接池（ConnectionPool）   │  │
│  │  复用 TCP 连接（5个/5分钟）     │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
              ↓ HTTP 请求
┌─────────────────────────────────────┐
│            Server                    │
└─────────────────────────────────────┘
```

---

## 2. OkHttp 核心架构

### 2.1 拦截链（Interceptors）

**什么是拦截链？**

拦截器按照顺序处理请求和响应，每个拦截器可以：
1. 修改请求（添加 Header、参数等）
2. 处理响应（解析、缓存、日志等）
3. 决定是否继续传递（短路返回）

**拦截器执行顺序：**

```
应用拦截器（Application Interceptors）
    ↓
RetryAndFollowUpInterceptor（重试和重定向）
    ↓
BridgeInterceptor（桥接：处理 Content-Type、编码等）
    ↓
CacheInterceptor（缓存处理）
    ↓
ConnectInterceptor（获取连接）
    ↓
网络拦截器（Network Interceptors）
    ↓
CallServerInterceptor（真正执行 HTTP 请求）
```

**代码示例：**

```kotlin
// 1. 日志拦截器（应用层）
.addInterceptor(HttpLoggingInterceptor().apply {
    level = if (BuildConfig.DEBUG) {
        HttpLoggingInterceptor.Level.BODY  // 打印请求/响应体
    } else {
        HttpLoggingInterceptor.Level.NONE
    }
})

// 2. 认证拦截器（应用层）
.addInterceptor { chain ->
    val request = chain.request().newBuilder()
        .header("Authorization", "Bearer $token")
        .build()
    chain.proceed(request)
}

// 3. 重试拦截器（应用层）
.addInterceptor { chain ->
    var request = chain.request()
    var response = chain.proceed(request)
    var tryCount = 0
    
    while (!response.isSuccessful && tryCount < 3) {
        tryCount++
        response.close()
        response = chain.proceed(request)
    }
    response
}

// 4. Chucker（应用层调试工具）
.addInterceptor(ChuckerInterceptor.Builder(context).build())
```

**应用拦截器 vs 网络拦截器区别：**

| 特性 | 应用拦截器 | 网络拦截器 |
|------|-----------|-----------|
| 调用时机 | 请求发送前/响应返回后 | 准备发送网络请求/收到网络响应 |
| 能看到 | 原始请求 | 最终请求（可能经过重定向） |
| 短路线程 | 可以 | 可以 |
| 使用场景 | 日志、认证、通用 Header | 查看最终请求内容 |

### 2.2 分发器（Dispatcher）

**职责：**
- 管理并发请求数量
- 分配线程执行请求
- 维护等待队列

**核心参数：**

```kotlin
val dispatcher = Dispatcher().apply {
    maxRequests = 64              // 最大并发请求数（默认64）
    maxRequestsPerHost = 5        // 每个主机最大并发数（默认5）
    executorService = ThreadPoolExecutor(
        0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS,
        SynchronousQueue(),
        threadFactory("OkHttp Dispatcher", false)
    )
}

OkHttpClient.Builder()
    .dispatcher(dispatcher)
    .build()
```

**执行流程：**

```
请求1 请求2 请求3 请求4 请求5 ... 请求100
  ↓     ↓     ↓     ↓     ↓         ↓
┌─────────────────────────────────────┐
│           等待队列                   │
│  (当并发数达到 maxRequests 时等待)   │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│         正在执行队列                  │
│  (最多 maxRequests 个同时执行)        │
└─────────────────────────────────────┘
              ↓
        线程池执行
```

**面试话术：**
> "Dispatcher 管理并发请求，默认最大 64 个并发，每个主机最多 5 个并发。对于音乐类 App，我会适当增大 per-host 并发数，因为我们需要同时加载封面图片、音频列表和用户信息。"

### 2.3 连接池（ConnectionPool）

**核心作用：**
1. 复用 TCP 连接（避免重复三次握手）
2. 管理连接生命周期（定期清理空闲连接）
3. 限制资源占用（防止内存泄漏）

**核心参数：**

```kotlin
val connectionPool = ConnectionPool(
    maxIdleConnections = 5,       // 最大空闲连接数（默认5）
    keepAliveDuration = 5,        // 连接保持时间（分钟，默认5）
    timeUnit = TimeUnit.MINUTES
)

OkHttpClient.Builder()
    .connectionPool(connectionPool)
    .build()
```

**TCP 连接复用原理：**

```
不复用的情况：
请求图片1: [三次握手] → 传输 → [四次挥手]  
请求图片2: [三次握手] → 传输 → [四次挥手]
请求图片3: [三次握手] → 传输 → [四次挥手]
总耗时: 300ms 握手 + 传输时间

复用的情况（ConnectionPool）：
请求图片1: [三次握手] → 传输 → [放回连接池]  
请求图片2: [从池子取] → 传输 → [放回连接池]
请求图片3: [从池子取] → 传输 → [放回连接池]
总耗时: 100ms 握手 + 传输时间 ✅ 省了 200ms！
```

**HTTP 版本差异：**

```kotlin
// HTTP/1.1 (Keep-Alive)
// 一个 TCP 连接同一时间只能处理一个请求（串行）
// 如果要并行，需要多个 TCP 连接
Connection: keep-alive

// HTTP/2 (多路复用)
// 一个 TCP 连接可以同时处理多个请求（并行）
// 不会出现队头阻塞
// 默认情况下 OkHttp 会自动协商升级到 HTTP/2
```

**面试话术：**
> "ConnectionPool 默认维护 5 个空闲连接，保持 5 分钟。复用连接可以避免重复的 TCP 三次握手，提升请求速度。对于音乐 App，我会增大到 10 个连接，因为我们有图片、音频、API 等多种请求需要并行处理。"

---

## 3. Retrofit 核心机制

### 3.1 动态代理原理

**什么是动态代理？**

在运行时动态创建一个实现接口的类，无需编译时编写实现代码。

**Retrofit 的工作流程：**

```kotlin
// 1. 定义接口
interface UserApi {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): User
}

// 2. 创建 Retrofit 实例
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build()

// 3. 生成接口实现（动态代理）
val api = retrofit.create(UserApi::class.java)

// 4. 调用时实际执行的是代理的 invoke 方法
val user = api.getUser("123")
//     ↓
// 代理拦截 → 解析注解 → 构建 Request → OkHttp 执行 → 转换结果
```

**内部简化伪代码：**

```kotlin
// Retrofit.create() 实际做的事：
Proxy.newProxyInstance(
    classLoader,
    arrayOf(UserApi::class.java)
) { proxy, method, args ->
    // 1. 从缓存获取或创建 ServiceMethod
    val serviceMethod = loadServiceMethod(method)
    
    // 2. 解析注解，构建 OkHttp Request
    val request = serviceMethod.toRequest(args)
    
    // 3. 调用 OkHttp 执行请求
    val response = okHttpClient.newCall(request).execute()
    
    // 4. Converter 转换响应数据
    serviceMethod.parseResponse(response)
}
```

**ServiceMethod 缓存（性能优化）：**

```kotlin
// Retrofit 内部缓存（避免重复反射解析）
private val serviceMethodCache = ConcurrentHashMap<Method, ServiceMethod<*>>()

// 第一次调用：解析注解 → 创建 ServiceMethod → 存入缓存
// 第二次调用：直接从缓存取（O(1) 复杂度）
```

### 3.2 Converter 数据转换

**职责：**
- 请求时：Kotlin 对象 → JSON/Protobuf/其他格式
- 响应时：ResponseBody → Kotlin 对象

**常见 Converter：**

```kotlin
// 1. Gson 转换器
.addConverterFactory(GsonConverterFactory.create())

// 2. Kotlinx Serialization 转换器（推荐）
.addConverterFactory(
    Json { ignoreUnknownKeys = true }
        .asConverterFactory("application/json".toMediaType())
)

// 3. Jackson 转换器
.addConverterFactory(JacksonConverterFactory.create())

// 4. Protobuf 转换器
.addConverterFactory(ProtoConverterFactory.create())
```

**转换流程：**

```
请求转换：
User(id="1", name="张三")
    ↓
GsonConverterFactory
    ↓
{"id":"1","name":"张三"}
    ↓
RequestBody

响应转换：
ResponseBody
    ↓
GsonConverterFactory
    ↓
{"id":"1","name":"张三"}
    ↓
User(id="1", name="张三")
```

### 3.3 CallAdapter 调用适配

**职责：** 适配不同的返回值类型

```kotlin
// 1. 默认支持（suspend 函数）
@GET("users/{id}")
suspend fun getUser(@Path("id") id: String): User

// 2. RxJava 支持
@GET("users/{id}")
fun getUser(@Path("id") id: String): Observable<User>

// 3. LiveData 支持
@GET("users/{id}")
fun getUser(@Path("id") id: String): LiveData<User>

// 添加 RxJava 适配器
.addCallAdapterFactory(RxJava3CallAdapterFactory.create())
```

---

## 4. 完整请求流程图解

```
┌──────────────────────────────────────────────────────────────────┐
│                           Application                             │
├──────────────────────────────────────────────────────────────────┤
│  1. 调用 API                                                      │
│     api.getUser("123")                                            │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│                           Retrofit                                │
├──────────────────────────────────────────────────────────────────┤
│  2. 动态代理拦截                                                   │
│     Proxy.invoke()                                                │
│                                                                   │
│  3. 获取 ServiceMethod（从缓存或解析）                              │
│     ├─ 解析 @GET/@Path 等注解                                       │
│     ├─ 确定 Converter（Gson/Kotlinx Serialization）                 │
│     └─ 确定 CallAdapter（suspend/Observable）                       │
│                                                                   │
│  4. 构建 OkHttp Request                                           │
│     Request.Builder()                                             │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│                           OkHttp                                  │
├──────────────────────────────────────────────────────────────────┤
│  5. 拦截链处理                                                     │
│     ├─ 应用拦截器：日志、认证、重试                                   │
│     ├─ RetryAndFollowUpInterceptor：自动重试和跟随重定向              │
│     ├─ BridgeInterceptor：处理 Content-Type、编码                   │
│     ├─ CacheInterceptor：根据 Cache-Control 处理缓存                 │
│     ├─ ConnectInterceptor：从 ConnectionPool 获取连接                │
│     ├─ 网络拦截器：查看最终请求                                       │
│     └─ CallServerInterceptor：执行 HTTP 请求                        │
│                                                                   │
│  6. Dispatcher 调度                                                │
│     ├─ 检查并发数（maxRequests=64）                                  │
│     ├─ 检查同主机并发（maxRequestsPerHost=5）                         │
│     └─ 分配线程执行                                                 │
│                                                                   │
│  7. ConnectionPool 管理连接                                         │
│     ├─ 检查是否有可复用的连接（keep-alive）                          │
│     ├─ 没有则创建新连接（三次握手）                                   │
│     └─ 使用后放回连接池或关闭                                        │
└──────────────────────────────────────────────────────────────────┘
                              ↓ HTTP/HTTPS
┌──────────────────────────────────────────────────────────────────┐
│                            Server                                 │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│                           OkHttp                                  │
├──────────────────────────────────────────────────────────────────┤
│  8. 接收响应                                                       │
│     Response（包含 ResponseBody）                                  │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│                           Retrofit                                │
├──────────────────────────────────────────────────────────────────┤
│  9. Converter 转换数据                                             │
│     ResponseBody → JSON String → User 对象                         │
│                                                                   │
│  10. 返回结果                                                      │
│      User(id="123", name="张三")                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 5. 项目实战配置分析

### 当前项目配置

```kotlin
// NetworkModule.kt（当前配置）
@Provides
@Singleton
fun providesOkHttpClient(): OkHttpClient {
    return OkHttpClient.Builder()
        // 超时配置
        .connectTimeout(10, TimeUnit.SECONDS)
        .writeTimeout(10, TimeUnit.SECONDS)
        .readTimeout(10, TimeUnit.SECONDS)
        
        // 日志拦截器（仅在 DEBUG 模式）
        .addInterceptor(HttpLoggingInterceptor().apply {
            setLevel(if (Config.DEBUG) Level.BODY else Level.NONE)
        })
        
        // Chucker 调试拦截器
        .addInterceptor(ChuckerInterceptor.Builder(context).build())
        
        // 认证拦截器（添加 Token）
        .addInterceptor { chain ->
            val request = chain.request().newBuilder()
                .header("Authorization", token)
                .build()
            chain.proceed(request)
        }
        
        // ❌ 没有设置 ConnectionPool（使用默认值：5个连接，5分钟）
        // ❌ 没有设置 Dispatcher（使用默认值：64并发，5 per host）
        .build()
}
```

### 优化建议配置

```kotlin
// 针对音乐类 App 的优化配置
@Provides
@Singleton
fun providesOkHttpClient(): OkHttpClient {
    return OkHttpClient.Builder()
        // 超时配置（保持默认或适当调整）
        .connectTimeout(10, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)    // 上传文件需要更长
        .readTimeout(30, TimeUnit.SECONDS)     // 下载音频需要更长
        
        // 连接池优化（增大空闲连接数）
        .connectionPool(
            ConnectionPool(
                maxIdleConnections = 10,       // 从 5 增大到 10
                keepAliveDuration = 5,         // 保持 5 分钟
                TimeUnit.MINUTES
            )
        )
        
        // 分发器优化（增大同主机并发）
        .dispatcher(
            Dispatcher().apply {
                maxRequests = 64               // 保持默认
                maxRequestsPerHost = 10        // 从 5 增大到 10
            }
        )
        
        // 拦截器链（保持原有）
        .addInterceptor(HttpLoggingInterceptor().apply {
            setLevel(if (Config.DEBUG) Level.BODY else Level.NONE)
        })
        .addInterceptor(ChuckerInterceptor.Builder(context).build())
        .addInterceptor(AuthInterceptor())
        
        .build()
}
```

**优化理由：**
1. **ConnectionPool(10, 5min)**：音乐 App 需要同时加载图片、音频元数据、用户数据，增大空闲连接数可以复用更多连接
2. **maxRequestsPerHost = 10**：同一个 CDN 服务器可能有多个并发请求（图片列表、音频列表）
3. **write/readTimeout = 30s**：上传/下载大文件（音频）需要更长时间

---

## 6. 性能优化与面试话术

### 面试核心问题准备

**Q1: OkHttp 的核心组件有哪些？**

> "OkHttp 核心有三个部分：拦截链（Interceptors）、分发器（Dispatcher）和连接池（ConnectionPool）。
>
> 拦截链按照顺序处理请求，包括应用拦截器（日志、认证）和系统拦截器（重试、缓存、连接）。
>
> 分发器管理并发请求，默认最大 64 个并发，每个主机 5 个并发。
>
> 连接池复用 TCP 连接，默认维护 5 个空闲连接，保持 5 分钟，避免重复三次握手。"

**Q2: Retrofit 和 OkHttp 的关系是什么？**

> "Retrofit 是基于 OkHttp 的封装框架。Retrofit 负责：
> 1. 通过注解定义 API 接口
> 2. 使用动态代理生成接口实现
> 3. Converter 进行数据转换（JSON ↔ 对象）
> 
> 真正的 HTTP 请求还是交给 OkHttp 执行。简单说：Retrofit 是声明式 API 层，OkHttp 是执行层。"

**Q3: 连接池的作用是什么？参数如何设置？**

> "连接池的作用是复用 TCP 连接，避免每次请求都进行三次握手，提升性能。
>
> 核心参数：
> - maxIdleConnections：最大空闲连接数（默认 5）
> - keepAliveDuration：连接保持时间（默认 5 分钟）
>
> 对于音乐类 App，我会设置为 ConnectionPool(10, 5, MINUTES)，因为我们需要同时加载封面图片、音频列表、用户信息，增大到 10 个可以复用更多连接。"

**Q4: 分发器的作用是什么？如何优化？**

> "分发器管理并发请求的调度和执行。核心参数：
> - maxRequests：最大并发请求数（默认 64）
> - maxRequestsPerHost：每个主机最大并发数（默认 5）
>
> 优化建议：对于音乐 App，我会把 maxRequestsPerHost 从 5 增大到 10，因为我们从同一个 CDN 服务器加载多个资源（图片、音频）时需要更高的并发。"

**Q5: Retrofit 的动态代理是怎么工作的？**

> "Retrofit 使用 Java 动态代理（Proxy.newProxyInstance）在运行时生成接口的实现类。
>
> 当调用接口方法时，代理的 invoke 方法会拦截调用，然后：
> 1. 从缓存获取 ServiceMethod（避免重复反射解析注解）
> 2. 根据注解构建 OkHttp Request
> 3. 调用 OkHttp 执行请求
> 4. 使用 Converter 转换响应数据
> 5. 返回给调用方
>
> ServiceMethod 会缓存解析结果，第一次调用后就不重复反射，提升性能。"

**Q6: Converter 和 CallAdapter 的区别？**

> "Converter 负责数据格式的转换，比如 JSON ↔ Kotlin 对象，使用 Gson 或 Kotlinx Serialization。
>
> CallAdapter 负责适配返回值类型，比如把 Call<User> 适配为 suspend 函数返回 User，或适配为 RxJava 的 Observable。
>
> 简单说：Converter 处理数据内容，CallAdapter 处理调用方式。"

### 常见优化手段总结

```kotlin
// 1. 连接池优化
.connectionPool(ConnectionPool(10, 5, TimeUnit.MINUTES))

// 2. 分发器优化
.dispatcher(Dispatcher().apply {
    maxRequestsPerHost = 10
})

// 3. 超时优化
.connectTimeout(10, TimeUnit.SECONDS)
.writeTimeout(30, TimeUnit.SECONDS)  // 上传大文件
.readTimeout(30, TimeUnit.SECONDS)   // 下载大文件

// 4. 拦截器优化（添加缓存）
.addInterceptor(CacheInterceptor(cache))

// 5. 协议优化（启用 HTTP/2）
// OkHttp 默认支持 HTTP/2，无需额外配置
.protocols(listOf(Protocol.HTTP_2, Protocol.HTTP_1_1))
```

---

## 7. 常见误区与正解

### 误区 1：ConnectionPool 管理线程 ❌

```kotlin
// ❌ 错误理解：
// "连接池从里面取线程来发 HTTP"

// ✅ 正解：
// ConnectionPool 管理的是 TCP Socket 连接
// Dispatcher 管理的是线程和并发请求

ConnectionPool(maxIdleConnections = 5)  // 5 个 TCP 连接
Dispatcher().apply {
    maxRequests = 64                     // 64 个并发请求（线程）
}
```

### 误区 2：Retrofit 设置 OkHttp 参数 ❌

```kotlin
// ❌ 错误理解：
// "Retrofit 可以设置连接池、分发器"

// ✅ 正解：
// Retrofit 只负责使用配置好的 OkHttpClient
// 参数是在构建 OkHttpClient 时设置的

val okHttpClient = OkHttpClient.Builder()
    .connectionPool(pool)      // 设置连接池
    .dispatcher(dispatcher)    // 设置分发器
    .build()

val retrofit = Retrofit.Builder()
    .client(okHttpClient)      // 传入配置好的 client
    .baseUrl(BASE_URL)
    .build()
```

### 误区 3：HTTP/1.1 连接可以并行处理多个请求 ❌

```kotlin
// ❌ 错误理解：
// "一个 TCP 连接可以同时处理多个请求"

// ✅ 正解：
// HTTP/1.1：一个连接同一时间只能处理一个请求（串行）
// HTTP/2：一个连接可以并行处理多个请求（多路复用）

// HTTP/1.1 需要多个连接才能并行
// HTTP/2 一个连接就够了
```

### 误区 4：拦截器一定会执行 ❌

```kotlin
// ❌ 错误理解：
// "添加了拦截器就一定会执行"

// ✅ 正解：
// 如果从缓存命中，可能不经过网络拦截器
// 如果短路返回（不调用 chain.proceed()），后续拦截器不会执行

.addInterceptor { chain ->
    if (shouldCache(chain.request())) {
        return cachedResponse  // 短路返回，后续拦截器不执行
    }
    chain.proceed(chain.request())
}
```

### 误区 5：动态代理每次调用都反射 ❌

```kotlin
// ❌ 错误理解：
// "动态代理每次调用都用反射解析注解，性能差"

// ✅ 正解：
// Retrofit 会缓存 ServiceMethod，第一次调用后从缓存取
// 性能接近直接调用

private val serviceMethodCache = ConcurrentHashMap<Method, ServiceMethod<*>>()

// 第一次：反射解析 → 存入缓存
// 第二次：从缓存取（O(1)）
```

---

## 总结

### 核心口诀

> **OkHttp 三剑客：拦截链、分发器、连接池**  
> **Retrofit 三板斧：动态代理、Converter、CallAdapter**  
> **连接复用省握手，并发控制防过载**  
> **注解定义 API，动态代理来执行**

### 面试黄金回答模板

```
1. OkHttp 核心组件：
   - 拦截链（日志、认证、缓存、重试）
   - 分发器（并发控制，默认64并发5 per host）
   - 连接池（TCP复用，默认5个空闲连接5分钟）

2. Retrofit 核心机制：
   - 注解定义 REST API
   - 动态代理生成实现
   - Converter 数据转换
   - ServiceMethod 缓存优化

3. 项目优化（音乐App）：
   - ConnectionPool(10, 5min)：复用更多连接
   - maxRequestsPerHost=10：提升同主机并发
   - write/readTimeout=30s：支持大文件传输
```

---

**创建时间**: 2025-03-19  
**标签**: OkHttp, Retrofit, Android, 网络请求, 性能优化, 面试
**版本**: 基于 OkHttp 4.12.0 + Retrofit 2.9.0