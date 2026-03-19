# Retrofit 网络请求框架学习笔记

## 目录

- [1. Retrofit 简介](#1-retrofit-简介)
  - [1.1 为什么要用 Retrofit](#11-为什么要用-retrofit)
  - [1.2 Retrofit 核心特点](#12-retrofit-核心特点)
  - [1.3 与传统方式对比](#13-与传统方式对比)
- [2. 快速入门](#2-快速入门)
  - [2.1 添加依赖](#21-添加依赖)
  - [2.2 最简单的 GET 请求](#22-最简单的-get-请求)
  - [2.3 最简单的 POST 请求](#23-最简单的-post-请求)
  - [2.4 常用注解说明](#24-常用注解说明)
- [3. 项目实战示例](#3-项目实战示例)
  - [3.1 完整架构介绍](#31-完整架构介绍)
  - [3.2 定义 API 接口](#32-定义-api-接口)
  - [3.3 统一响应封装](#33-统一响应封装)
  - [3.4 配置 OkHttp 拦截器](#34-配置-okhttp-拦截器)
  - [3.5 结合 Hilt 依赖注入](#35-结合-hilt-依赖注入)
  - [3.6 DataSource 层实现](#36-datasource-层实现)
- [4. 底层原理](#4-底层原理)
  - [4.1 动态代理机制](#41-动态代理机制)
  - [4.2 请求构建流程](#42-请求构建流程)
  - [4.3 数据转换流程](#43-数据转换流程)
  - [4.4 执行流程图解](#44-执行流程图解)
- [5. 常见问题与最佳实践](#5-常见问题与最佳实践)
  - [5.1 常见错误处理](#51-常见错误处理)
  - [5.2 性能优化建议](#52-性能优化建议)

---

## 1. Retrofit 简介

### 1.1 为什么要用 Retrofit

在 Android 开发中，网络请求是必不可少的功能。没有 Retrofit 之前，我们可能需要这样写：

```kotlin
// ❌ 传统方式：手写 HTTP 请求
val url = URL("https://api.example.com/users/1")
val connection = url.openConnection() as HttpURLConnection
connection.requestMethod = "GET"
connection.setRequestProperty("Content-Type", "application/json")

val responseCode = connection.responseCode
if (responseCode == 200) {
    val reader = BufferedReader(InputStreamReader(connection.inputStream))
    val response = reader.readText()
    // 手动解析 JSON
    val user = JSONObject(response)
    // ... 还要处理线程切换、错误处理等
}
```

**Retrofit 带来的好处：**

| 优势 | 说明 |
|------|------|
| **声明式 API** | 通过接口+注解定义请求，代码清晰易读 |
| **自动序列化** | 自动将 JSON 转换为 Kotlin 对象 |
| **线程管理** | 自动处理后台线程与主线程切换 |
| **解耦架构** | 网络层与业务层分离，易于维护和测试 |
| **扩展性强** | 支持自定义拦截器、Converter、Adapter |

### 1.2 Retrofit 核心特点

1. **类型安全**：编译时检查 API 定义，避免运行时错误
2. **协程支持**：原生支持 Kotlin 协程和 `suspend` 函数
3. **多种转换器**：支持 Gson、Jackson、Kotlinx Serialization 等
4. **OkHttp 基础**：基于 OkHttp，继承其所有特性（连接池、拦截器等）
5. **动态代理**：运行时生成接口实现，无需手动编写网络代码

### 1.3 与传统方式对比

| 特性 | HttpURLConnection | Retrofit |
|------|-------------------|----------|
| 代码量 | 多（手动处理所有细节） | 少（声明式 API） |
| 可读性 | 差 | 好 |
| 线程管理 | 手动处理 | 自动处理 |
| JSON 解析 | 手动解析 | 自动转换 |
| 错误处理 | 手动编写 | 统一处理 |
| 扩展性 | 差 | 强 |

---

## 2. 快速入门

### 2.1 添加依赖

```kotlin
// build.gradle.kts (Module: app)
dependencies {
    // Retrofit 核心库
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    
    // JSON 转换器（可选其一）
    // Gson 转换器
    implementation("com.squareup.retrofit2:converter-gson:2.9.0")
    // 或 Kotlinx Serialization 转换器
    implementation("com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter:1.0.0")
    
    // OkHttp（Retrofit 基于 OkHttp）
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    // OkHttp 日志拦截器（调试用）
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
}
```

### 2.2 最简单的 GET 请求

**步骤 1：定义数据模型**

```kotlin
// 用户数据模型
// 使用 Kotlinx Serialization
import kotlinx.serialization.Serializable

@Serializable
data class User(
    val id: String,
    val name: String,
    val email: String,
    val avatar: String? = null
)

// 统一响应封装
@Serializable
data class ApiResponse<T>(
    val data: T? = null,
    val status: Int = 0,
    val message: String? = null
) {
    val isSuccess: Boolean get() = status == 0
}
```

**步骤 2：定义 API 接口**

```kotlin
import retrofit2.http.GET
import retrofit2.http.Path
import retrofit2.http.Query

interface UserApiService {
    
    // GET 请求：获取用户列表
    // URL: https://api.example.com/users?page=1&size=10
    @GET("users")
    suspend fun getUsers(
        @Query("page") page: Int,
        @Query("size") size: Int
    ): ApiResponse<List<User>>
    
    // GET 请求：获取单个用户
    // URL: https://api.example.com/users/123
    @GET("users/{id}")
    suspend fun getUser(@Path("id") userId: String): ApiResponse<User>
    
    // GET 请求：搜索用户
    @GET("users/search")
    suspend fun searchUsers(@Query("keyword") keyword: String): ApiResponse<List<User>>
}
```

**步骤 3：创建 Retrofit 实例并调用**

```kotlin
import retrofit2.Retrofit
import com.jakewharton.retrofit2.converter.kotlinx.serialization.asConverterFactory
import kotlinx.serialization.json.Json
import okhttp3.MediaType.Companion.toMediaType

class RetrofitClient {
    
    private val retrofit = Retrofit.Builder()
        .baseUrl("https://api.example.com/")  // 基础 URL
        .addConverterFactory(
            Json { ignoreUnknownKeys = true }
                .asConverterFactory("application/json".toMediaType())
        )
        .build()
    
    val userApi: UserApiService = retrofit.create(UserApiService::class.java)
}

// 使用示例
suspend fun fetchUsers() {
    val client = RetrofitClient()
    
    try {
        val response = client.userApi.getUsers(page = 1, size = 10)
        
        if (response.isSuccess) {
            val users = response.data
            println("获取到 ${users?.size} 个用户")
        } else {
            println("请求失败: ${response.message}")
        }
    } catch (e: Exception) {
        println("网络错误: ${e.message}")
    }
}
```

### 2.3 最简单的 POST 请求

```kotlin
import retrofit2.http.Body
import retrofit2.http.POST

// 登录请求数据
@Serializable
data class LoginRequest(
    val phone: String,
    val password: String
)

// 登录响应数据
@Serializable
data class LoginResponse(
    val token: String,
    val user: User
)

interface AuthApiService {
    
    // POST 请求：用户登录
    @POST("auth/login")
    suspend fun login(@Body request: LoginRequest): ApiResponse<LoginResponse>
    
    // POST 请求：用户注册
    @POST("auth/register")
    suspend fun register(@Body user: User): ApiResponse<User>
}

// 使用示例
suspend fun doLogin(phone: String, password: String) {
    val client = RetrofitClient()
    val authApi = retrofit.create(AuthApiService::class.java)
    
    val request = LoginRequest(phone, password)
    val response = authApi.login(request)
    
    if (response.isSuccess) {
        val token = response.data?.token
        // 保存 token 到本地
        println("登录成功，Token: $token")
    }
}
```

### 2.4 常用注解说明

| 注解 | 用途 | 示例 |
|------|------|------|
| `@GET` | 发送 GET 请求 | `@GET("users")` |
| `@POST` | 发送 POST 请求 | `@POST("users")` |
| `@PUT` | 发送 PUT 请求 | `@PUT("users/{id}")` |
| `@DELETE` | 发送 DELETE 请求 | `@DELETE("users/{id}")` |
| `@Path` | URL 路径参数 | `@Path("id") userId: String` |
| `@Query` | URL 查询参数 | `@Query("page") page: Int` |
| `@QueryMap` | 多个查询参数 | `@QueryMap params: Map<String, String>` |
| `@Body` | 请求体（POST/PUT） | `@Body user: User` |
| `@Header` | 请求头 | `@Header("Authorization") token: String` |
| `@Headers` | 固定请求头 | `@Headers("Content-Type: application/json")` |
| `@Multipart` | 多部分请求（文件上传） | `@Multipart` |
| `@Part` | 多部分中的一个部分 | `@Part file: MultipartBody.Part` |

---

## 3. 项目实战示例

### 3.1 完整架构介绍

在实际项目中，Retrofit 通常配合以下架构组件使用：

```
UI Layer (Activity/Fragment/ViewModel)
    ↓ 调用
Repository Layer (UserRepository)
    ↓ 调用
DataSource Layer (MyRetrofitDataSource)
    ↓ 调用
Network Layer (Retrofit + OkHttp)
    ↓ HTTP 请求
Server (后端 API)
```

**依赖注入关系：**
```
NetworkModule (Hilt Module)
    ├─ provides OkHttpClient (带拦截器)
    ├─ provides Json (序列化配置)
    └─ provides Call.Factory

MyRetrofitDataSource
    ├─ 注入: networkJson
    ├─ 注入: okhttpCallFactory
    └─ 创建 Retrofit 实例

ViewModel
    └─ 注入: MyRetrofitDataSource
```

### 3.2 定义 API 接口

```kotlin
package com.quick.app.core.network.retrofit

import retrofit2.http.*

/**
 * 网络请求接口定义
 * 
 * 技巧：
 * 1. 按业务模块分组（region 注释）
 * 2. 使用 suspend 函数支持协程
 * 3. 统一返回 NetworkResponse<T> 格式
 * 4. 参数使用具名参数，提高可读性
 */
interface MyNetworkApiService {
    
    //region 音乐模块
    @GET("v1/songs/page")
    suspend fun songs(): NetworkResponse<NetworkPageData<Song>>

    @GET("v1/songs/info")
    suspend fun songDetail(
        @Query(value = "id") id: String,
    ): NetworkResponse<Song>
    //endregion

    //region 用户模块
    /**
     * 用户登录
     * @param data 用户信息（手机号/密码）
     */
    @POST("v1/login")
    suspend fun login(
        @Body data: User,
    ): NetworkResponse<Session>

    /**
     * 微信登录
     * @param data 微信授权码
     */
    @POST("v2/wechat-login")
    suspend fun loginWechat(
        @Body data: WechatLoginRequest
    ): NetworkResponse<Session>

    /**
     * 获取用户详情
     * @param id 用户ID
     */
    @GET("v1/users/info")
    suspend fun userDetail(@Query(value = "id") id: String): NetworkResponse<User>

    /**
     * 更新用户信息
     * @param data 用户数据
     */
    @POST("v1/users/update")
    suspend fun updateUser(@Body data: User): NetworkResponse<BaseModel>
    //endregion

    //region 歌单模块
    /**
     * 获取用户创建的歌单
     * @param userId 用户ID（URL 路径参数）
     */
    @GET("v1/users/{userId}/create")
    suspend fun createSheets(
        @Path("userId") userId: String
    ): NetworkResponse<NetworkPageData<Sheet>>

    /**
     * 创建歌单
     * @param data 歌单信息
     */
    @POST("v1/sheets/add")
    suspend fun createSheet(@Body data: Sheet): NetworkResponse<Sheet>

    /**
     * 收藏歌单
     * @param data 歌单ID
     */
    @POST("v1/collects/add")
    suspend fun collectSheet(@Body data: BaseId): NetworkResponse<BaseModel>
    //endregion

    //region 文件上传
    /**
     * 上传文件（多部分表单）
     * @param file 文件部分
     * @param flavor 渠道参数
     * @param relative 是否返回相对路径
     */
    @Multipart
    @POST("v1/r")
    suspend fun uploadFile(
        @Part file: MultipartBody.Part,
        @Part("flavor") flavor: RequestBody,
        @Part("relative") relative: RequestBody,
    ): NetworkResponse<BaseId>

    /**
     * 批量上传文件
     * @param files 多个文件
     */
    @Multipart
    @POST("v1/r/batch")
    suspend fun uploadFiles(
        @Part files: List<MultipartBody.Part>,
        @Part("flavor") flavor: RequestBody,
        @Part("relative") relative: RequestBody,
    ): NetworkResponse<NetworkPageData<String>>
    //endregion

    //region 内容模块
    /**
     * 获取内容列表（多个查询参数）
     * @param last 最后一条记录ID（分页用）
     * @param categoryId 分类ID
     * @param userId 用户ID
     * @param size 每页数量
     * @param style 样式类型
     * @param current 当前页
     */
    @GET("v1/contents/page")
    suspend fun contents(
        @Query(value = "last") last: String?,
        @Query(value = "category_id") categoryId: String?,
        @Query(value = "user_id") userId: String?,
        @Query(value = "size") size: Int,
        @Query(value = "style") style: Int? = null,
        @Query(value = "current") current: String? = null,
    ): NetworkResponse<NetworkPageData<Content>>

    /**
     * 获取动态列表（使用 QueryMap 简化参数）
     * @param data 查询参数映射
     */
    @GET("v1/feeds/page")
    suspend fun feeds(@QueryMap data: Map<String, String>): NetworkResponse<NetworkPageData<Feed>>
    //endregion

    //region 搜索模块
    @GET("v1/searches/sheets")
    suspend fun searchSheets(@Query(value = "query") query: String): NetworkResponse<NetworkPageData<Sheet>>

    @GET("v1/searches/suggests")
    suspend fun searchSuggest(@Query(value = "query") query: String): NetworkResponse<Suggest>
    //endregion
}
```

### 3.3 统一响应封装

```kotlin
package com.quick.app.core.model.response

import kotlinx.serialization.Serializable

/**
 * 统一网络响应格式
 * 
 * 后端返回的统一格式：
 * {
 *   "data": T,        // 业务数据（泛型）
 *   "status": 0,      // 状态码，0 表示成功
 *   "message": null   // 错误信息
 * }
 */
@Serializable
data class NetworkResponse<T>(
    /** 真实数据，类型是泛型 */
    val data: T? = null,
    
    /** 状态码，等于 0 表示成功 */
    val status: Int = 0,
    
    /** 错误提示信息 */
    val message: String? = null,
) {
    /** 是否成功 */
    val isSucceeded: Boolean
        get() = status == 0
}

/**
 * 分页数据结构
 */
@Serializable
data class NetworkPageData<T>(
    /** 数据列表 */
    val data: List<T> = emptyList(),
    
    /** 当前页码 */
    val page: Int = 1,
    
    /** 每页数量 */
    val size: Int = 10,
    
    /** 总记录数 */
    val total: Long = 0,
    
    /** 是否有下一页 */
    val hasNext: Boolean = false
)
```

### 3.4 配置 OkHttp 拦截器

```kotlin
package com.quick.app.core.network.di

import android.util.Log
import com.chuckerteam.chucker.api.ChuckerInterceptor
import com.quick.app.MyAppState
import com.quick.app.MyApplication
import com.quick.app.core.config.Config
import com.quick.app.util.Constant
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import kotlinx.serialization.json.Json
import okhttp3.Call
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import java.util.concurrent.TimeUnit
import javax.inject.Singleton

/**
 * 网络依赖注入模块
 * 
 * 职责：
 * 1. 提供 Json 序列化配置
 * 2. 配置 OkHttpClient（拦截器、超时等）
 * 3. 提供 Call.Factory 供 Retrofit 使用
 */
@Module
@InstallIn(SingletonComponent::class)
class NetworkModule {

    /**
     * 提供 Json 配置
     * ignoreUnknownKeys = true：忽略未知的 JSON 字段，防止解析失败
     */
    @Provides
    @Singleton
    fun providesNetworkJson(): Json = Json {
        ignoreUnknownKeys = true
    }

    /**
     * 提供 Call.Factory（OkHttpClient）
     */
    @Provides
    @Singleton
    fun okHttpCallFactory(okHttpClient: OkHttpClient): Call.Factory = okHttpClient

    /**
     * 配置 OkHttpClient
     * 
     * 添加的拦截器（按顺序执行）：
     * 1. HttpLoggingInterceptor：日志输出（调试用）
     * 2. ChuckerInterceptor：应用内显示网络请求信息
     * 3. 自定义拦截器：添加认证头
     */
    @Provides
    @Singleton
    fun providesOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            // 超时配置
            .connectTimeout(10, TimeUnit.SECONDS)   // 连接超时
            .writeTimeout(10, TimeUnit.SECONDS)     // 写入超时
            .readTimeout(10, TimeUnit.SECONDS)      // 读取超时
            
            // 日志拦截器（仅在 DEBUG 模式启用）
            .addInterceptor(HttpLoggingInterceptor().apply {
                setLevel(
                    if (Config.DEBUG)
                        HttpLoggingInterceptor.Level.BODY  // 打印请求/响应体
                    else
                        HttpLoggingInterceptor.Level.NONE
                )
            })
            
            // Chucker 拦截器（应用内调试工具）
            .addInterceptor(
                ChuckerInterceptor.Builder(MyApplication.instance).build()
            )
            
            // 自定义拦截器：添加认证 Token
            .addInterceptor { chain ->
                var request = chain.request()
                
                // 如果已登录，添加 Authorization Header
                if (MyAppState.session.isNotBlank()) {
                    Log.d(TAG, "添加认证头: ${MyAppState.session}")
                    
                    request = request.newBuilder()
                        .header(Constant.HEADER_AUTH, MyAppState.session)
                        .build()
                }
                
                chain.proceed(request)
            }
            .build()
    }

    companion object {
        const val TAG = "NetworkModule"
    }
}
```

### 3.5 结合 Hilt 依赖注入

```kotlin
package com.quick.app.core.network.di

import com.quick.app.core.config.Config
import com.quick.app.core.network.datasource.MyRetrofitDatasource
import com.quick.app.core.network.datasource.MyNetworkDatasource
import dagger.Binds
import dagger.Module
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent

/**
 * 带 flavor 的网络模块
 * 
 * 使用 @Binds 将接口绑定到实现类
 * Hilt 会自动注入 MyRetrofitDatasource 实例
 */
@Module
@InstallIn(SingletonComponent::class)
abstract class FlavoredNetworkModule {
    
    @Binds
    abstract fun bindsNetworkDataSource(
        myRetrofitDatasource: MyRetrofitDatasource
    ): MyNetworkDatasource
}
```

**在 ViewModel 中使用：**

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val networkDataSource: MyNetworkDatasource
) : ViewModel() {

    private val _user = MutableStateFlow<User?>(null)
    val user: StateFlow<User?> = _user.asStateFlow()

    fun loadUser(userId: String) {
        viewModelScope.launch {
            try {
                val response = networkDataSource.userDetail(userId)
                if (response.isSucceeded) {
                    _user.value = response.data
                }
            } catch (e: Exception) {
                // 处理错误
            }
        }
    }
}
```

### 3.6 DataSource 层实现

```kotlin
package com.quick.app.core.network.datasource

import com.jakewharton.retrofit2.converter.kotlinx.serialization.asConverterFactory
import com.quick.app.core.config.Config
import com.quick.app.core.model.*
import com.quick.app.core.model.response.*
import com.quick.app.core.network.retrofit.MyNetworkApiService
import kotlinx.serialization.json.Json
import okhttp3.Call
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.MultipartBody
import okhttp3.RequestBody
import retrofit2.Retrofit
import javax.inject.Inject

/**
 * Retrofit 数据源实现
 * 
 * 职责：
 * 1. 创建 Retrofit 实例
 * 2. 实现 MyNetworkDatasource 接口
 * 3. 将 API 调用转发给 MyNetworkApiService
 * 
 * 为什么要多这一层？
 * - 隔离 Retrofit 具体实现，方便切换其他网络库
 * - 统一处理网络异常
 * - 便于单元测试（可以 mock 这一层）
 */
class MyRetrofitDatasource @Inject constructor(
    networkJson: Json,
    okhttpCallFactory: Call.Factory,
) : MyNetworkDatasource {

    /**
     * 创建 Retrofit 实例
     */
    private val service = Retrofit.Builder()
        .baseUrl(Config.ENDPOINT)                           // 基础 URL
        .callFactory(okhttpCallFactory)                      // 使用自定义 OkHttpClient
        .addConverterFactory(
            networkJson.asConverterFactory("application/json".toMediaType())
        )                                                   // 添加 Kotlinx Serialization 转换器
        .build()
        .create(MyNetworkApiService::class.java)            // 创建 API 接口实现

    //region 音乐相关
    override suspend fun songs(): NetworkResponse<NetworkPageData<Song>> {
        return service.songs()
    }

    override suspend fun songDetail(id: String): NetworkResponse<Song> {
        return service.songDetail(id)
    }
    //endregion

    //region 用户相关
    override suspend fun login(data: User): NetworkResponse<Session> {
        return service.login(data)
    }

    override suspend fun loginWechat(data: WechatLoginRequest): NetworkResponse<Session> {
        return service.loginWechat(data)
    }

    override suspend fun userDetail(id: String): NetworkResponse<User> {
        return service.userDetail(id)
    }

    override suspend fun updateUser(data: User): NetworkResponse<BaseModel> {
        return service.updateUser(data)
    }
    //endregion

    //region 歌单相关
    override suspend fun sheetDetail(id: String): NetworkResponse<Sheet> {
        return service.sheetDetail(id)
    }

    override suspend fun createSheets(userId: String): NetworkResponse<NetworkPageData<Sheet>> {
        return service.createSheets(userId)
    }

    override suspend fun collectSheet(data: BaseId): NetworkResponse<BaseModel> {
        return service.collectSheet(data)
    }

    override suspend fun cancelCollectSheet(data: BaseId): NetworkResponse<BaseModel> {
        return service.cancelCollectSheet(data)
    }
    //endregion

    //region 文件上传
    override suspend fun uploadFile(
        file: MultipartBody.Part,
        flavor: RequestBody,
        relative: RequestBody
    ): NetworkResponse<BaseId> {
        return service.uploadFile(file, flavor, relative)
    }

    override suspend fun uploadFiles(
        files: List<MultipartBody.Part>,
        flavor: RequestBody,
        relative: RequestBody
    ): NetworkResponse<NetworkPageData<String>> {
        return service.uploadFiles(files, flavor, relative)
    }
    //endregion

    // 其他方法实现...
}
```

**文件上传示例：**

```kotlin
/**
 * 上传头像
 */
suspend fun uploadAvatar(context: Context, uri: Uri): String? {
    // 1. 创建 MultipartBody.Part
    val inputStream = context.contentResolver.openInputStream(uri)
    val file = inputStream?.readBytes() ?: return null
    
    val requestFile = file.toRequestBody("image/*".toMediaTypeOrNull())
    val filePart = MultipartBody.Part.createFormData(
        "file", 
        "avatar.jpg", 
        requestFile
    )
    
    // 2. 创建其他参数
    val flavor = "prod".toRequestBody("text/plain".toMediaTypeOrNull())
    val relative = "0".toRequestBody("text/plain".toMediaTypeOrNull())
    
    // 3. 调用上传接口
    val response = networkDataSource.uploadFile(filePart, flavor, relative)
    
    return if (response.isSucceeded) {
        response.data?.id  // 返回文件 URL
    } else null
}
```

---

## 4. 底层原理

### 4.1 动态代理机制

Retrofit 最核心的技术是**动态代理**（Dynamic Proxy）。

**什么是动态代理？**

动态代理允许你在运行时创建一个实现了一组接口的类，而无需在编译时编写实现代码。

**Retrofit 的工作方式：**

```kotlin
// 1. 你定义一个接口
interface UserApi {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): User
}

// 2. Retrofit 使用动态代理创建实现
val api = retrofit.create(UserApi::class.java)

// 3. 当你调用方法时，实际上是调用代理的 invoke
val user = api.getUser("123")
//     ↓
// 代理拦截这个调用，解析注解，构建请求，执行网络调用
```

**核心源码简析：**

```kotlin
// Retrofit.java（简化版）
public <T> T create(final Class<T> service) {
    return Proxy.newProxyInstance(
        service.getClassLoader(),
        new Class<?>[] { service },
        new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) {
                // 1. 解析方法注解（@GET, @POST, @Path 等）
                ServiceMethod serviceMethod = loadServiceMethod(method);
                
                // 2. 构建 OkHttp 请求
                Request request = serviceMethod.toRequest(args);
                
                // 3. 调用 OkHttp 执行请求
                okhttp3.Call call = callFactory.newCall(request);
                
                // 4. 解析响应并转换数据
                return serviceMethod.parseResponse(call.execute());
            }
        }
    );
}
```

### 4.2 请求构建流程

当你调用 API 方法时，Retrofit 内部经历了以下步骤：

```
调用 api.getUser("123")
    ↓
代理拦截方法调用
    ↓
解析方法注解
    ├─ @GET("users/{id}") → 请求方法 + URL 路径
    ├─ @Path("id") → 替换 URL 中的 {id} 为参数值
    └─ 返回值类型 User → 确定响应转换器
    ↓
构建 OkHttp Request 对象
    ├─ URL: https://api.example.com/users/123
    ├─ Method: GET
    ├─ Headers: Content-Type: application/json
    └─ Body: null
    ↓
调用 OkHttpClient 执行请求
    ↓
获取 Response
    ↓
Converter 转换 JSON → Kotlin 对象
    ↓
返回 User 对象
```

### 4.3 数据转换流程

**请求转换（Request）：**

```kotlin
// Kotlin 对象 → JSON
@Body data: User
    ↓
Kotlinx Serialization Converter
    ↓
{"id": "1", "name": "张三", "email": "zhangsan@example.com"}
    ↓
RequestBody
```

**响应转换（Response）：**

```kotlin
// JSON → Kotlin 对象
HTTP Response Body:
{"status": 0, "data": {"id": "1", "name": "张三"}, "message": null}
    ↓
ResponseBody.string()
    ↓
Kotlinx Serialization Converter
    ↓
NetworkResponse<User>(
    status = 0,
    data = User(id = "1", name = "张三"),
    message = null
)
```

**转换器链：**

```
接口返回值: NetworkResponse<User>
    ↓
CallAdapter：处理 suspend 函数，包装为协程
    ↓
Converter：JSON ↔ Kotlin 对象
    ↓
实际返回: User 对象（或 Response 包装）
```

### 4.4 执行流程图解

```
┌─────────────────────────────────────────────────────────────────┐
│                          Application                             │
├─────────────────────────────────────────────────────────────────┤
│  ViewModel                                                       │
│    ├─ 调用 repository.fetchData()                               │
│    └─ 在协程中执行（viewModelScope）                             │
├─────────────────────────────────────────────────────────────────┤
│  Repository                                                      │
│    └─ 调用 dataSource.fetchData()                               │
├─────────────────────────────────────────────────────────────────┤
│  MyRetrofitDatasource                                            │
│    ├─ 持有 MyNetworkApiService 接口                              │
│    └─ 调用 service.fetchData()                                  │
├─────────────────────────────────────────────────────────────────┤
│                          Retrofit                                │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 动态代理 (Proxy)                                          │  │
│  │   └─ invoke() 拦截方法调用                                 │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │ ServiceMethod（缓存的方法解析）                            │  │
│  │   ├─ 解析注解 → HTTP 方法、URL、参数                       │  │
│  │   └─ 解析返回类型 → 确定 Converter                        │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │ RequestBuilder（构建 OkHttp Request）                     │  │
│  │   └─ URL、Headers、Body 组装                              │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                          OkHttp                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 拦截器链 (Interceptor Chain)                              │  │
│  │   ├─ 日志拦截器                                            │  │
│  │   ├─ Chucker 拦截器                                       │  │
│  │   └─ 认证拦截器（添加 Token）                              │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │ 连接池管理（复用 TCP 连接）                                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│    ↓ HTTP/HTTPS 请求                                            │
├─────────────────────────────────────────────────────────────────┤
│                          Server                                  │
└─────────────────────────────────────────────────────────────────┘
```

**完整调用链：**

```
suspend fun getUser(id: String): User
    ↓
Retrofit 动态代理拦截
    ↓
ServiceMethod.parseAnnotations() 解析注解
    ├─ HTTP Method: GET
    ├─ URL Path: users/{id} → users/123
    └─ Return Type: User
    ↓
RequestFactory.create(args) 创建请求
    ↓
OkHttpCall.execute() 执行请求
    ↓
OkHttp Interceptors 拦截器处理
    ↓
ResponseBody 解析
    ↓
Converter.responseBodyConverter() JSON → User
    ↓
返回 User 对象
```

---

## 5. 常见问题与最佳实践

### 5.1 常见错误处理

**1. 网络异常统一处理**

```kotlin
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Error(val code: Int, val message: String) : NetworkResult<Nothing>()
    object Loading : NetworkResult<Nothing>()
}

suspend fun <T> safeApiCall(apiCall: suspend () -> NetworkResponse<T>): NetworkResult<T> {
    return try {
        val response = apiCall()
        if (response.isSucceeded) {
            NetworkResult.Success(response.data!!)
        } else {
            NetworkResult.Error(response.status, response.message ?: "未知错误")
        }
    } catch (e: IOException) {
        NetworkResult.Error(-1, "网络连接失败")
    } catch (e: HttpException) {
        NetworkResult.Error(e.code(), e.message ?: "HTTP 错误")
    } catch (e: Exception) {
        NetworkResult.Error(-2, "发生错误: ${e.message}")
    }
}

// 使用
viewModelScope.launch {
    when (val result = safeApiCall { api.getUser("123") }) {
        is NetworkResult.Success -> { /* 处理成功 */ }
        is NetworkResult.Error -> { /* 处理错误 */ }
        NetworkResult.Loading -> { /* 显示加载 */ }
    }
}
```

**2. Token 过期自动刷新**

```kotlin
class AuthInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val response = chain.proceed(request)
        
        if (response.code == 401) { // Token 过期
            // 同步刷新 Token
            val newToken = refreshTokenSync()
            
            // 重试原请求
            val newRequest = request.newBuilder()
                .header("Authorization", "Bearer $newToken")
                .build()
            return chain.proceed(newRequest)
        }
        
        return response
    }
}
```

### 5.2 性能优化建议

**1. 单例 Retrofit 实例**

```kotlin
// ❌ 错误：每次调用都创建新实例
fun createApi(): ApiService {
    return Retrofit.Builder()...build().create(ApiService::class.java)
}

// ✅ 正确：使用单例模式或依赖注入
@Singleton
class ApiProvider @Inject constructor(retrofit: Retrofit) {
    val api: ApiService = retrofit.create(ApiService::class.java)
}
```

**2. 启用 OkHttp 连接池**

```kotlin
OkHttpClient.Builder()
    .connectionPool(ConnectionPool(10, 5, TimeUnit.MINUTES))
    .build()
```

**3. 合理设置超时**

```kotlin
OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS)   // 连接超时（短）
    .writeTimeout(30, TimeUnit.SECONDS)     // 写入超时（文件上传需要长）
    .readTimeout(10, TimeUnit.SECONDS)      // 读取超时
    .build()
```

**4. 缓存 GET 请求**

```kotlin
// 添加缓存拦截器
val cacheSize = 10 * 1024 * 1024L // 10MB
val cache = Cache(context.cacheDir, cacheSize)

OkHttpClient.Builder()
    .cache(cache)
    .addInterceptor { chain ->
        var request = chain.request()
        if (!isNetworkAvailable()) {
            // 无网络时使用缓存
            request = request.newBuilder()
                .cacheControl(CacheControl.FORCE_CACHE)
                .build()
        }
        chain.proceed(request)
    }
    .build()
```

**5. 使用 @JvmSuppressWildcards 避免泛型问题**

```kotlin
// 如果接口使用泛型列表，可能需要添加
@POST("users/batch")
suspend fun createUsers(
    @Body users: List<@JvmSuppressWildcards User>
): ApiResponse<Void>
```

---

## 总结

**Retrofit 核心要点：**

1. **声明式编程**：接口 + 注解定义 API，代码清晰简洁
2. **自动转换**：JSON ↔ Kotlin 对象自动转换
3. **协程支持**：原生支持 suspend 函数，自动线程管理
4. **扩展性强**：支持自定义 Converter、Adapter、Interceptor
5. **基于 OkHttp**：继承所有 OkHttp 特性（连接池、拦截器等）

**学习路径建议：**

```
Step 1: 掌握基本使用（GET/POST + 简单参数）
    ↓
Step 2: 学习高级特性（文件上传、拦截器）
    ↓
Step 3: 结合项目架构（Repository + DataSource + DI）
    ↓
Step 4: 理解底层原理（动态代理、转换流程）
    ↓
Step 5: 性能优化与最佳实践
```

**推荐资源：**

- [Retrofit 官方文档](https://square.github.io/retrofit/)
- [OkHttp 官方文档](https://square.github.io/okhttp/)
- [Kotlinx Serialization](https://github.com/Kotlin/kotlinx.serialization)

---

**创建时间**: 2025-03-19  
**标签**: Retrofit, OkHttp, Android, 网络请求, Kotlin
**版本**: 基于 Retrofit 2.9.0