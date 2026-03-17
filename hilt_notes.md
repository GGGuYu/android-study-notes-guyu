# Hilt 依赖注入笔记

## 1. 什么是 Hilt

Android 的依赖注入框架，类似 Spring 的 DI。核心思想：**不要自己 new，让框架创建和管理对象**。

---

## 2. 核心注解

### @Inject - 标记需要注入的构造函数

```kotlin
// 类直接注入（不需要 @Binds）
class SessionRepository @Inject constructor(
    private val networkDataSource: MyNetworkDatasource  // Hilt 自动注入
)

// ViewModel 注入
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val sessionRepository: SessionRepository,     // 类，直接注入
    private val userDataRepository: UserDataRepository    // 接口，需要 @Binds
) : ViewModel()
```

### @Module + @Binds - 接口绑定实现类

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class DataModule {
    
    @Binds
    @Singleton  // 标记单例（可选）
    abstract fun bindsUserDataRepository(
        userDataRepository: LocalUserDataRepository  // 实现类
    ): UserDataRepository                             // 接口
}
```

**要点**：
- 只有**接口**需要 `@Binds`，**类**直接 `@Inject` 即可
- `@Module` 必须是 `abstract class`
- 方法必须是 `abstract`

---

## 3. @Binds vs @Provides

| 场景 | 使用 | 示例 |
|------|------|------|
| 接口 → 实现类 | `@Binds` | 自己的 Repository 接口 |
| 需要创建逻辑 | `@Provides` | 第三方库、需要计算参数 |

```kotlin
// @Binds：接口绑定（更简洁）
@Binds
abstract fun bindsUserDataRepository(
    impl: LocalUserDataRepository
): UserDataRepository

// @Provides：需要创建逻辑
@Provides
fun provideMediaServiceConnection(
    @ApplicationContext context: Context
): MediaServiceConnection {
    return MediaServiceConnection.getInstance(context)  // 有创建逻辑
}
```

---

## 4. 默认不是单例！

**重要**：默认情况下每次注入都创建新实例，需要显式标记 `@Singleton`。

```kotlin
// 不是单例！每次注入都 new 一个
@Provides
fun provideMediaServiceConnection(...): MediaServiceConnection

// 是单例！整个 App 生命周期只有一个
@Provides
@Singleton
fun provideMediaServiceConnection(...): MediaServiceConnection

// @Binds 同理
@Binds
@Singleton  // 必须显式标记
abstract fun bindsUserDataRepository(...)
```

---

## 5. 依赖嵌套自动解决

Hilt 会自动处理依赖链：

```kotlin
// ViewModel 需要 Repository
class LoginViewModel @Inject constructor(
    private val userDataRepository: UserDataRepository
)

// Repository 需要 DataSource
class LocalUserDataRepository @Inject constructor(
    private val myPreferencesDatasource: MyPreferencesDatasource
)

// DataSource 需要 DataStore
class MyPreferencesDatasource @Inject constructor(
    private val userPreferences: DataStore<UserDataPreferences>
)
```

**创建流程**：
```
创建 LoginViewModel
    → 需要 UserDataRepository（接口）
        → 查 @Binds，找到 LocalUserDataRepository
            → 需要 MyPreferencesDatasource
                → 需要 DataStore
                    → DataStore 创建成功
                → 注入 DataStore ✓
            → MyPreferencesDatasource 创建成功
        → 注入 MyPreferencesDatasource ✓
    → LocalUserDataRepository 创建成功
→ 注入 LocalUserDataRepository ✓
→ LoginViewModel 创建成功
```

---

## 6. 作用域（生命周期）

| 注解 | 生命周期 | 说明 |
|------|---------|------|
| `@Singleton` | 整个 App | 应用不销毁就一直存在 |
| `@ActivityScoped` | Activity | Activity 存活期间 |
| `@ViewModelScoped` | ViewModel | ViewModel 存活期间 |

```kotlin
@Module
@InstallIn(SingletonComponent::class)  // 作用域：整个 App
abstract class DataModule { ... }
```

---

## 7. 快速记忆

```
接口 → @Binds 告诉 Hilt 用哪个实现
类   → 直接 @Inject constructor，Hilt 自己 new
单例 → 必须显式 @Singleton，默认每次都 new
```

---

## 8. 面试常问

**Q: Hilt 和 Dagger 的关系？**  
A: Hilt 是基于 Dagger 的封装，预定义好了 Component，使用更简单。

**Q: 为什么用 @Binds 不用 @Provides？**  
A: @Binds 更简洁，专用于接口绑定实现类，性能更好。

**Q: Hilt 怎么实现单例？**  
A: 编译时生成 Double Check 单例模式代码，不是运行时反射。

**Q: 默认是单例吗？**  
A: 不是！必须显式标记 @Singleton。
