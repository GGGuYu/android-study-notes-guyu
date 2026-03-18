# Android Service 学习笔记

## 我的简洁理解

### Service 就两种类型

**1. 非绑定式（启动型）**：默认就是
- 用 `startService()` 启动
- `onCreate()` 只触发一次，`onStartCommand()` 每次被 start 都执行，用来发命令
- 和非绑定式 Service 通信是**单向**的（通过 Intent）
- 销毁只能用 `stopService()` 或 `stopSelf()`
- 独立运行，不依赖 Activity

**2. 绑定式**：
- 用 `bindService()` 启动
- 生命周期有 `onBind()`、`onUnbind`
- 跟随**绑定者**走（所有绑定者解绑后自动销毁），比非绑定式多了一种销毁方式
- 通过返回 **IBinder** 让 Activity 访问 Service 的信息，实现**双向**通信
- 可以有多个组件同时绑定

**3. 前台服务**：
- 不是第三种类型，是 Service 的一种**运行状态**
- **启动型和绑定型都能变成前台服务**
- 必须挂一个 **Notification 通知栏**
- 启动后 **5 秒内**必须调用 `startForeground()`，否则 ANR

---

## 核心概念详解

### 1. Service 的两种基本类型

#### 1.1 启动型 Service（Started Service）

```kotlin
// Activity 中启动
startService(Intent(this, MusicService::class.java))
```

**生命周期**：
```
onCreate() → onStartCommand() → [运行中] → onDestroy()
     ↑            ↑（每次start都执行）           ↓
     └──── 只执行一次 ──────────────────────────┘
```

**特点**：
- 独立运行，**不依赖任何 Activity**
- 即使启动它的 Activity 销毁了，Service 继续运行
- 必须手动调用 `stopService()` 或 `stopSelf()` 销毁
- 通信是**单向的**：Activity → Service（通过 Intent）

#### 1.2 绑定型 Service（Bound Service）

```kotlin
// 1. 定义 ServiceConnection（关键！）
private val connection = object : ServiceConnection {
    override fun onServiceConnected(name: ComponentName, service: IBinder) {
        // 绑定成功后，在这里拿到 IBinder
        val binder = service as MusicService.MusicBinder
        musicService = binder.getService()  // 赋值给 Activity 变量
    }
    
    override fun onServiceDisconnected(name: ComponentName) {
        musicService = null  // Service 异常断开时清理
    }
}

// 2. 发起绑定
bindService(intent, connection, Context.BIND_AUTO_CREATE)

// 3. 解绑（防止内存泄漏）
override fun onDestroy() {
    super.onDestroy()
    if (isBound) {
        unbindService(connection)
    }
}
```

**生命周期**：
```
onCreate() → onBind() → [运行中] → onUnbind() → onDestroy()
                              ↓
                    所有绑定者都解绑后自动销毁
```

**特点**：
- 通过 `IBinder` 实现**双向通信**（Activity ↔ Service）
- **跟随绑定者生命周期**（所有绑定者解绑后自动销毁）
- 可以有多个组件同时绑定

### 2. 双向通信的实现

**Service 端**：
```kotlin
class MusicService : Service() {
    private val binder = MusicBinder()
    
    // 内部类暴露 Service 的方法和属性
    inner class MusicBinder : Binder() {
        fun getService(): MusicService = this@MusicService
        
        // 暴露具体功能
        fun play() { /*...*/ }
        fun pause() { /*...*/ }
        fun getCurrentSong(): Song? { /*...*/ }
    }
    
    override fun onBind(intent: Intent): IBinder = binder
    
    // Service 内部的数据和方法
    private var currentSong: Song? = null
    fun setSong(song: Song) { currentSong = song }
}
```

**Activity 端**：
```kotlin
class PlayerActivity : AppCompatActivity() {
    private var musicService: MusicService? = null
    
    private val connection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName, service: IBinder) {
            val binder = service as MusicService.MusicBinder
            musicService = binder.getService()  // 获取 Service 实例
            
            // 现在可以直接调用 Service 的方法了！
            binder.play()
            val song = binder.getCurrentSong()
        }
        
        override fun onServiceDisconnected(name: ComponentName) {
            musicService = null
        }
    }
}
```

### 3. 前台 Service（Foreground Service）

**⚠️ 关键纠正**：前台服务**不是第三种 Service 类型**，而是一种运行状态！

```kotlin
class MusicService : Service() {
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // ✅ 必须在启动后 5 秒内调用 startForeground()！
        startForeground(NOTIFICATION_ID, createNotification())
        return START_STICKY
    }
    
    private fun createNotification(): Notification {
        // Android 8.0+ 必须创建 NotificationChannel
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                CHANNEL_ID,
                "音乐播放",
                NotificationManager.IMPORTANCE_LOW
            )
            getSystemService(NotificationManager::class.java)
                .createNotificationChannel(channel)
        }
        
        return NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("正在播放")
            .setContentText("歌曲名")
            .setSmallIcon(R.drawable.ic_music)
            .build()
    }
}
```

**启动前台服务**：
```kotlin
// Android 8.0+ 使用 startForegroundService()
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    startForegroundService(intent)
} else {
    startService(intent)
}
```

**AndroidManifest.xml 配置**：
```xml
<!-- 必须声明权限 -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />

<!-- Android 10+ 需要声明 foregroundServiceType -->
<service 
    android:name=".MusicService"
    android:foregroundServiceType="mediaPlayback" />
```

**前台服务的特点**：

| 特点 | 说明 |
|------|------|
| 显示通知 | 必须显示在通知栏，用户可见 |
| 高优先级 | 系统不会轻易杀死（内存不足时） |
| 时间限制 | **必须在 5 秒内调用 startForeground()**，否则 ANR |
| 续航影响 | 用户能看到应用在后台运行 |
| 权限要求 | Android 9+ 需要 FOREGROUND_SERVICE 权限 |

## 对比总结

| 特性 | 启动型 Service | 绑定型 Service |
|------|---------------|---------------|
| 启动方式 | `startService()` | `bindService()` |
| 停止方式 | `stopService()` / `stopSelf()` | 自动（所有绑定者解绑） |
| 生命周期 | 独立运行 | 跟随绑定者 |
| 通信方式 | 单向（Intent） | 双向（IBinder） |
| 能否前台 | ✅ 可以 | ✅ 可以 |
| 典型场景 | 后台下载、音乐播放 | 需要实时交互（如控制播放器） |

## 常见错误理解

❌ **Service 有三种类型**（启动型、绑定型、前台型）  
✅ 只有两种基本类型，前台是运行状态

❌ **必须在 onCreate() 中调用 startForeground()**  
✅ 可以在 onCreate() 或 onStartCommand() 中，但必须在 5 秒内完成

❌ **bindService() 立即返回 IBinder**  
✅ 是异步的，通过 ServiceConnection.onServiceConnected() 回调获取

❌ **绑定型 Service 只能绑定一个 Activity**  
✅ 可以有多个组件同时绑定，所有解绑后才销毁

## 一句话总结

> **启动型 Service** 是"独行侠"，独立运行、单向通信；**绑定型 Service** 是"合伙人"，跟随绑定者、双向交互；**前台 Service** 是给 Service 加了个"护身符"（通知），让系统别杀掉它。

---

记录日期：2026年3月18日  
学习内容：Service 的类型、生命周期、通信机制、前台服务
