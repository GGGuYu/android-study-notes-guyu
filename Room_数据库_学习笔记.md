# Room 数据库学习笔记

## 目录

- [1. 一句话理解](#1-一句话理解)
- [2. 快速复习卡片](#2-快速复习卡片)
- [3. 三个核心组件](#3-三个核心组件)
- [4. 最简单的使用示例](#4-最简单的使用示例)
- [5. 关键特性](#5-关键特性)
- [6. 与 Compose 集成](#6-与-compose-集成)
- [7. 常见面试题](#7-常见面试题)

---

## 1. 一句话理解

**Room = SQLite 的"智能包装"**

不用写繁琐的 SQL 语句和游标代码，用注解就能操作数据库，还能自动把数据变成 Kotlin 对象。

```
传统 SQLite：写 SQL → 执行 → 手动解析 Cursor → 变成对象（很啰嗦）
Room：定义实体类 → 写注解 → 直接拿到对象（很简单）
```

---

## 2. 快速复习卡片

| 关键词 | 一句话理解 |
|--------|-----------|
| **@Entity** | 标记这个类是数据库表 |
| **@Dao** | 数据库操作接口（增删改查） |
| **@Database** | 数据库总入口，包含所有表 |
| **@Query** | 写 SQL 查询语句 |
| **@Insert** | 插入数据 |
| **@Update** | 更新数据 |
| **@Delete** | 删除数据 |
| **@PrimaryKey** | 主键（唯一标识） |
| **Flow** | 数据变化自动通知（不用手动刷新） |
| **Migration** | 数据库升级（版本变更） |
| **TypeConverter** | 把复杂类型（如 Date）转成数据库能存的类型 |
| **@Transaction** | 多个操作一起执行，要么都成功要么都失败 |

---

## 3. 三个核心组件

Room 只需要写三个东西：

### 3.1 实体类（Entity）= 表结构

```kotlin
// 告诉 Room：这个类对应数据库里的 "users" 表
@Entity(tableName = "users")
data class User(
    @PrimaryKey  // 主键：唯一标识这条数据
    val id: String,
    
    @ColumnInfo(name = "user_name")  // 数据库列名叫 user_name
    val name: String,
    
    val age: Int  // 不写注解，默认用属性名做列名
)
```

### 3.2 DAO（Data Access Object）= 操作接口

```kotlin
@Dao
interface UserDao {
    // 查询所有用户，返回 Flow（数据变化自动更新）
    @Query("SELECT * FROM users")
    fun getAllUsers(): Flow<List<User>>
    
    // 根据 ID 查一个用户
    @Query("SELECT * FROM users WHERE id = :userId")
    fun getUserById(userId: String): Flow<User>
    
    // 插入用户（suspend = 挂起函数，不卡主线程）
    @Insert
    suspend fun insertUser(user: User)
    
    // 插入多个用户
    @Insert
    suspend fun insertUsers(users: List<User>)
    
    // 更新用户
    @Update
    suspend fun updateUser(user: User)
    
    // 删除用户
    @Delete
    suspend fun deleteUser(user: User)
}
```

### 3.3 Database = 数据库总入口

```kotlin
// 告诉 Room：这是一个数据库，包含 users 表
@Database(
    entities = [User::class],  // 所有表都写在这里
    version = 1                // 数据库版本号
)
abstract class AppDatabase : RoomDatabase() {
    // 提供 DAO 的抽象方法
    abstract fun userDao(): UserDao
}
```

---

## 4. 最简单的使用示例

### 4.1 添加依赖

```kotlin
// build.gradle.kts
dependencies {
    // Room 核心
    implementation("androidx.room:room-runtime:2.6.1")
    kapt("androidx.room:room-compiler:2.6.1")
    
    // Kotlin 协程支持
    implementation("androidx.room:room-ktx:2.6.1")
}
```

### 4.2 创建数据库（Hilt 依赖注入）

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "my_database.db"  // 数据库文件名
        ).build()
    }
    
    @Provides
    fun provideUserDao(database: AppDatabase): UserDao {
        return database.userDao()
    }
}
```

### 4.3 在 ViewModel 中使用

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userDao: UserDao
) : ViewModel() {
    
    // 获取所有用户（Flow 自动更新）
    val allUsers: StateFlow<List<User>> = userDao.getAllUsers()
        .stateIn(
            scope = viewModelScope,
            initialValue = emptyList(),
            started = SharingStarted.WhileSubscribed(5000)
        )
    
    // 添加用户
    fun addUser(user: User) {
        viewModelScope.launch {
            userDao.insertUser(user)
        }
    }
    
    // 删除用户
    fun deleteUser(user: User) {
        viewModelScope.launch {
            userDao.deleteUser(user)
        }
    }
}
```

### 4.4 在 Compose 中显示

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    // 自动监听数据库变化
    val users by viewModel.allUsers.collectAsStateWithLifecycle()
    
    LazyColumn {
        items(users) { user ->
            Text("${user.name}, ${user.age}岁")
        }
    }
    
    // 添加按钮
    Button(onClick = {
        viewModel.addUser(User(id = "1", name = "张三", age = 25))
    }) {
        Text("添加用户")
    }
}
```

---

## 5. 关键特性

### 5.1 返回 Flow（自动更新）

```kotlin
// 关键特性：数据变化自动通知 UI
@Query("SELECT * FROM users")
fun getAllUsers(): Flow<List<User>>

// 原理：
// 1. Room 知道你在查 "users" 表
// 2. 当 users 表有增删改时，自动重新查询
// 3. Flow 发射新数据，UI 自动刷新
// 4. 不需要手动调用 refresh()！
```

### 5.2 TypeConverter（处理复杂类型）

```kotlin
// Room 只能存基本类型（Int, String等），Date 不行
// 需要转换器：Date -> Long（时间戳）

class Converters {
    @TypeConverter
    fun fromDate(date: Date?): Long? {
        return date?.time
    }
    
    @TypeConverter
    fun toDate(timestamp: Long?): Date? {
        return timestamp?.let { Date(it) }
    }
}

// 在 Database 上注册转换器
@Database(entities = [User::class], version = 1)
@TypeConverters(Converters::class)  // ← 加上这个
abstract class AppDatabase : RoomDatabase()
```

### 5.3 Migration（数据库升级）

```kotlin
// 场景：版本 1 升级到版本 2，给 users 表加一列 "email"

val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL(
            "ALTER TABLE users ADD COLUMN email TEXT DEFAULT ''"
        )
    }
}

// 使用 Migration
Room.databaseBuilder(context, AppDatabase::class.java, "db")
    .addMigrations(MIGRATION_1_2)  // ← 告诉 Room 怎么升级
    .build()
```

**注意**：如果不写 Migration，用户升级 App 会崩溃！

### 5.4 关系查询（一对多）

```kotlin
// 一个用户有多个歌单
@Entity
data class User(
    @PrimaryKey val userId: String,
    val name: String
)

@Entity
data class Playlist(
    @PrimaryKey val playlistId: String,
    val name: String,
    val userId: String  // 外键：关联到 User
)

// 查询用户和他的所有歌单
data class UserWithPlaylists(
    @Embedded val user: User,  // 用户基本信息
    
    @Relation(
        parentColumn = "userId",     // User 的主键
        entityColumn = "userId"      // Playlist 的外键
    )
    val playlists: List<Playlist>   // 关联的歌单列表
)

@Dao
interface UserDao {
    @Transaction  // ← 必须用事务
    @Query("SELECT * FROM users WHERE userId = :userId")
    fun getUserWithPlaylists(userId: String): UserWithPlaylists
}
```

### 5.5 事务（保证原子性）

```kotlin
// 场景：转账操作，扣款和加钱必须一起成功或一起失败

@Dao
interface AccountDao {
    @Transaction  // ← 加上这个
    suspend fun transfer(fromId: String, toId: String, amount: Double) {
        val from = getAccount(fromId)
        val to = getAccount(toId)
        
        updateAccount(from.copy(balance = from.balance - amount))
        updateAccount(to.copy(balance = to.balance + amount))
        // 要么都成功，要么都失败（自动回滚）
    }
}
```

---

## 6. 与 Compose 集成

Room + Flow + ViewModel + Compose = **数据变化自动刷新 UI**

```kotlin
// 数据流向：
数据库 (Room)
    ↓
DAO 返回 Flow<List<User>>
    ↓
ViewModel 转成 StateFlow
    ↓
Compose 用 collectAsStateWithLifecycle() 收集
    ↓
UI 自动更新

// 代码流程：
@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    fun getAllUsers(): Flow<List<User>>  // ← Room 自动观察变化
}

@HiltViewModel
class UserViewModel @Inject constructor(
    userDao: UserDao
) : ViewModel() {
    
    // 转成 StateFlow 供 Compose 使用
    val users = userDao.getAllUsers()
        .stateIn(
            scope = viewModelScope,
            initialValue = emptyList(),
            started = SharingStarted.WhileSubscribed(5000)
        )
}

@Composable
fun UserListScreen(viewModel: UserViewModel = hiltViewModel()) {
    val users by viewModel.users.collectAsStateWithLifecycle()
    
    LazyColumn {
        items(users) { user ->
            UserItem(user)
        }
    }
}

// 效果：
// 1. 在别处添加用户：userDao.insertUser(user)
// 2. Room 自动检测到 users 表变化
// 3. Flow 发射新数据
// 4. UI 自动刷新，显示新用户
// 全程不需要手动刷新！
```

---

## 7. 常见面试题

### Q1: Room 和直接使用 SQLite 有什么区别？

**简单回答：**
> "Room 是 SQLite 的包装，让我们不用写繁琐的 SQL 和 Cursor 代码。主要好处：
> 1. 用注解就能操作数据库，代码更简洁
> 2. 编译时检查 SQL 语句，不会运行时出错
> 3. 自动把数据库结果转成 Kotlin 对象
> 4. 支持 Flow，数据变化自动通知 UI"

### Q2: 数据库升级怎么做？

**简单回答：**
> "写 Migration 类，告诉 Room 怎么从旧版本升级到新版本。比如版本 1 到 2 要加一列：
> ```kotlin
> val MIGRATION_1_2 = object : Migration(1, 2) {
>     override fun migrate(db: SupportSQLiteDatabase) {
>         db.execSQL("ALTER TABLE users ADD COLUMN email TEXT")
>     }
> }
> ```
> 然后在 databaseBuilder 里用 addMigrations() 加上去。"

### Q3: Room 怎么支持 Date 类型？

**简单回答：**
> "Room 不支持 Date，要用 TypeConverter 转成 Long（时间戳）。写一个转换器类，两个方法：Date -> Long 和 Long -> Date，然后在 Database 上用 @TypeConverters 注册。"

### Q4: 为什么查询返回 Flow？

**简单回答：**
> "返回 Flow 可以让数据变成响应式的。Room 会监听你查询的表，当表数据变化时，自动重新查询并发射新数据。这样 UI 可以自动更新，不需要手动刷新。"

### Q5: @Transaction 是做什么的？

**简单回答：**
> "保证多个数据库操作要么都成功，要么都失败。比如转账：扣款和加钱必须一起成功，如果一个失败就自动回滚，不会出现钱扣了但没到账的情况。"

---

## 核心口诀

> **Room 三件套：Entity 表，DAO 操作，Database 总入口**  
> **注解省代码：@Query 查询，@Insert 插入，Flow 自动刷**  
> **升级用 Migration，复杂类型 TypeConverter**  
> **事务保安全，关系查询 @Relation**

---

**创建时间**: 2025-03-19  
**标签**: Room, SQLite, Android, 数据库, Flow, Compose