# Android 广播机制学习笔记

## 目录

1. [一句话总结](#一句话总结)
2. [广播类型对比](#广播类型对比)
3. [核心概念](#核心概念)
   - Intent Filter（过滤器）
   - 两种注册方式
4. [代码示例](#代码示例)
   - 无序广播（默认）
   - 有序广播
   - 本地广播
   - 粘性广播（已废弃）
   - 动态注册
   - 静态注册
5. [有序广播的应用场景](#有序广播的应用场景)
6. [广播 vs Activity 传参](#广播-vs-activity-传参)
7. [常见误区澄清](#常见误区澄清)

---

## 一句话总结

- **无序广播** = 群发消息，所有人同时收到，互不干扰
- **有序广播** = 排队处理，优先级高的先处理，可以截断或修改
- **本地广播** = 家庭内部群聊，只有自家 App 能听到
- **粘性广播** = 历史消息会保留，后注册的人也能收到之前发的（已废弃，了解即可）
- **静态注册** = 24小时待机，App 没启动也能收消息
- **动态注册** = 在岗期间监听，Activity 死了就自动失联

---

## 广播类型对比

| 类型 | 发送方式 | 特点 | 使用场景 |
|------|---------|------|---------|
| **无序广播** (Normal) | `sendBroadcast()` | 并行发送，无序，不可截断 | 普通通知，状态更新 |
| **有序广播** (Ordered) | `sendOrderedBroadcast()` | 串行发送，有优先级，可截断/修改 | 权限控制、拦截处理 |
| **本地广播** (Local) | `LocalBroadcastManager` | 仅 App 内部，安全高效 | App 内部组件通信 |
| **粘性广播** (Sticky) | `sendStickyBroadcast()` | 保留最后一条广播，新注册者能收到历史 | ⚠️ 已废弃，了解即可 |

---

## 核心概念

### Intent Filter（过滤器）

作用：**决定哪些接收器能收到广播**

```xml
<!-- 发送方和接收方都要声明 action -->
<intent-filter>
    <action android:name="com.example.DOWNLOAD_COMPLETE" />
    <action android:name="com.example.USER_LOGIN" />
</intent-filter>
```

**匹配规则**：
- 发送方的 Intent action 必须和接收方 filter 中的 action 一致
- 可以设置多个 action，匹配任意一个即可

### 两种注册方式

| 特性 | 动态注册 | 静态注册 |
|------|---------|---------|
| **注册位置** | 代码中 (`registerReceiver()`) | AndroidManifest.xml |
| **生命周期** | 跟随注册者（Activity/Service） | 独立存在 |
| **App 未启动** | ❌ 收不到 | ✅ 系统会启动 App 进程 |
| **适用场景** | 页面内的临时事件监听 | 系统级全局事件监听 |
| **示例** | 电量变化、屏幕旋转 | 开机启动、网络变化 |

---

## 代码示例

### 1. 无序广播（默认）

**发送方**：

```kotlin
// 发送简单广播
val intent = Intent("com.example.DOWNLOAD_COMPLETE")
intent.putExtra("file_path", "/downloads/song.mp3")
sendBroadcast(intent)
```

**接收方**（动态注册）：

```kotlin
class MainActivity : AppCompatActivity() {
    
    private lateinit var downloadReceiver: BroadcastReceiver
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 创建接收器
        downloadReceiver = object : BroadcastReceiver() {
            override fun onReceive(context: Context, intent: Intent) {
                val filePath = intent.getStringExtra("file_path")
                Toast.makeText(context, "下载完成: $filePath", Toast.LENGTH_SHORT).show()
            }
        }
        
        // 注册（必须在 onDestroy 注销！）
        val filter = IntentFilter("com.example.DOWNLOAD_COMPLETE")
        registerReceiver(downloadReceiver, filter)
    }
    
    override fun onDestroy() {
        super.onDestroy()
        unregisterReceiver(downloadReceiver)  // 必须注销，否则内存泄漏！
    }
}
```

---

### 2. 有序广播

**发送方**：

```kotlin
val intent = Intent("com.example.SMS_RECEIVED")
intent.putExtra("sms_body", "验证码: 123456")

// 发送有序广播
sendOrderedBroadcast(intent, null)  // 第二个参数是权限，null 表示不需要权限
```

**接收方 A**（高优先级，先收到）：

```xml
<receiver android:name=".receiver.SmsInterceptorReceiver">
    <intent-filter android:priority="1000">  <!-- 优先级最高 -->
        <action android:name="com.example.SMS_RECEIVED" />
    </intent-filter>
</receiver>
```

```kotlin
class SmsInterceptorReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val smsBody = intent.getStringExtra("sms_body")
        
        // 1. 读取原始数据
        Log.d("SmsInterceptor", "收到短信: $smsBody")
        
        // 2. 添加额外数据给下一个接收器
        resultData = "已处理 by SmsInterceptor"
        
        // 3. 如果是广告短信，截断广播（后面的接收器收不到）
        if (smsBody?.contains("广告") == true) {
            abortBroadcast()  // 截断！
            Log.d("SmsInterceptor", "广告短信已拦截")
        }
    }
}
```

**接收方 B**（低优先级，后收到）：

```xml
<receiver android:name=".receiver.SmsBackupReceiver">
    <intent-filter android:priority="100">  <!-- 优先级较低 -->
        <action android:name="com.example.SMS_RECEIVED" />
    </intent-filter>
</receiver>
```

```kotlin
class SmsBackupReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        // 获取上一个接收器添加的数据
        val processedData = resultData
        Log.d("SmsBackup", "上一个处理者留下: $processedData")
        
        // 备份短信到云端
        val smsBody = intent.getStringExtra("sms_body")
        backupToCloud(smsBody)
    }
}
```

**有序广播的关键 API**：

```kotlin
// 获取上一个接收器设置的结果数据
val data = resultData

// 设置结果数据给下一个接收器
resultData = "处理完成"

// 截断广播（后面的接收器收不到）
abortBroadcast()

// 检查是否被截断
val isAborted = isOrderedBroadcast && resultCode == Activity.RESULT_CANCELED
```

---

### 3. 本地广播

> ⚠️ **注意**：`LocalBroadcastManager` 已废弃，现在推荐使用 LiveData/Flow
> 但了解其原理有助于理解广播机制

```kotlin
// 获取本地广播管理器（已废弃，了解即可）
val localBroadcastManager = LocalBroadcastManager.getInstance(this)

// 发送本地广播
val intent = Intent("com.example.LOCAL_EVENT")
localBroadcastManager.sendBroadcast(intent)

// 注册本地广播接收器
val receiver = object : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        // 只有本 App 内的接收器能收到
    }
}
localBroadcastManager.registerReceiver(receiver, IntentFilter("com.example.LOCAL_EVENT"))
```

**现代替代方案**（推荐）：

```kotlin
// 使用 LiveData 替代本地广播
class MyViewModel : ViewModel() {
    private val _downloadEvent = MutableLiveData<String>()
    val downloadEvent: LiveData<String> = _downloadEvent
    
    fun notifyDownloadComplete(path: String) {
        _downloadEvent.value = path
    }
}
```

---

### 4. 粘性广播（已废弃，了解即可）

> ⚠️ **注意**：粘性广播（Sticky Broadcast）已在 API 21 (Android 5.0) 废弃，Google 不推荐使用了。
> 但在面试或看老代码时可能会遇到，所以简单了解。

**特点**：
- 系统会保留最后一条广播消息
- 后注册的接收器也能收到之前发送的广播
- 发送后不需要立即处理，新接收器注册时会自动收到最新的一条

**典型例子**（系统源码还在用）：
```kotlin
// 系统发送的电量广播就是粘性的
// 就算你的 App 刚启动，也能立即获取当前电量
val batteryIntent = registerReceiver(null, IntentFilter(Intent.ACTION_BATTERY_CHANGED))
val level = batteryIntent?.getIntExtra(BatteryManager.EXTRA_LEVEL, -1)
```

**为什么不推荐用**：
- 安全问题：任何应用都能读取粘性广播的内容
- 内存泄漏：广播会一直保存在系统中
- API 21 后废弃，随时可能移除

**现代替代方案**：
- 用 `LiveData`、`StateFlow` 保存状态
- 用 `SharedPreferences` 持久化
- 用 `Room` 数据库存储

---

### 5. 动态注册完整示例

```kotlin
class MusicPlayerActivity : AppCompatActivity() {
    
    private lateinit var headsetReceiver: BroadcastReceiver
    private lateinit var batteryReceiver: BroadcastReceiver
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_music_player)
        
        // 1. 耳机插拔监听
        headsetReceiver = object : BroadcastReceiver() {
            override fun onReceive(context: Context, intent: Intent) {
                if (intent.action == AudioManager.ACTION_AUDIO_BECOMING_NOISY) {
                    // 耳机拔出，暂停播放
                    pauseMusic()
                    Toast.makeText(context, "耳机已拔出，音乐暂停", Toast.LENGTH_SHORT).show()
                }
            }
        }
        registerReceiver(headsetReceiver, IntentFilter(AudioManager.ACTION_AUDIO_BECOMING_NOISY))
        
        // 2. 电量监听
        batteryReceiver = object : BroadcastReceiver() {
            override fun onReceive(context: Context, intent: Intent) {
                val level = intent.getIntExtra(BatteryManager.EXTRA_LEVEL, -1)
                val scale = intent.getIntExtra(BatteryManager.EXTRA_SCALE, -1)
                val batteryPct = level * 100 / scale.toFloat()
                
                if (batteryPct < 20) {
                    // 电量低提示
                    showLowBatteryWarning()
                }
            }
        }
        registerReceiver(batteryReceiver, IntentFilter(Intent.ACTION_BATTERY_CHANGED))
    }
    
    override fun onDestroy() {
        super.onDestroy()
        // 必须注销所有接收器！
        unregisterReceiver(headsetReceiver)
        unregisterReceiver(batteryReceiver)
    }
    
    private fun pauseMusic() { /* ... */ }
    private fun showLowBatteryWarning() { /* ... */ }
}
```

---

### 6. 静态注册完整示例

**AndroidManifest.xml**：

```xml
<!-- 开机启动接收器 -->
<receiver 
    android:name=".receiver.BootReceiver"
    android:enabled="true"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>

<!-- 网络变化接收器 -->
<receiver android:name=".receiver.NetworkReceiver">
    <intent-filter>
        <action android:name="android.net.conn.CONNECTIVITY_CHANGE" />
    </intent-filter>
</receiver>

<!-- 时区变化接收器 -->
<receiver android:name=".receiver.TimeZoneReceiver">
    <intent-filter>
        <action android:name="android.intent.action.TIMEZONE_CHANGED" />
        <action android:name="android.intent.action.TIME_SET" />
    </intent-filter>
</receiver>
```

**BootReceiver.kt**（开机启动）：

```kotlin
class BootReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED) {
            Log.d("BootReceiver", "手机开机了！")
            
            // 启动后台服务
            val serviceIntent = Intent(context, MusicPlaybackService::class.java)
            context.startForegroundService(serviceIntent)
            
            // 恢复之前的播放状态
            restorePlaybackState(context)
        }
    }
    
    private fun restorePlaybackState(context: Context) {
        // 读取 SharedPreferences，恢复播放列表
    }
}
```

**NetworkReceiver.kt**（网络变化）：

```kotlin
class NetworkReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        val networkInfo = connectivityManager.activeNetworkInfo
        
        if (networkInfo?.isConnected == true) {
            Log.d("NetworkReceiver", "网络已连接")
            
            // 有网了，同步离线数据
            syncOfflineData(context)
            
            // 更新音乐库
            checkForUpdates(context)
        } else {
            Log.d("NetworkReceiver", "网络已断开")
        }
    }
    
    private fun syncOfflineData(context: Context) { /* ... */ }
    private fun checkForUpdates(context: Context) { /* ... */ }
}
```

**所需权限**：

```xml
<!-- 开机启动权限 -->
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />

<!-- 网络状态权限 -->
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

---

## 有序广播的应用场景

### 场景 1：短信拦截器（权限控制）

```
接收器优先级：
┌─────────────────────────────────────┐
│ 1. 垃圾短信过滤器 (priority=1000)    │  ← 先收到，识别广告后截断
│    - 如果是广告 → abortBroadcast()  │
│    - 如果是正常短信 → 放行          │
├─────────────────────────────────────┤
│ 2. 验证码提取器 (priority=500)       │  ← 提取验证码，标记已处理
│    - 提取验证码并填充到输入框        │
│    - setResultData("已填充")        │
├─────────────────────────────────────┤
│ 3. 短信备份服务 (priority=100)       │  ← 最后收到，备份到云端
│    - 获取 resultData 判断是否已处理  │
└─────────────────────────────────────┘
```

### 场景 2：权限检查链

```kotlin
// 高优先级：权限检查
class PermissionCheckReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (!hasPermission(context)) {
            abortBroadcast()  // 没权限？直接截断！
            Toast.makeText(context, "无权执行此操作", Toast.LENGTH_SHORT).show()
        }
    }
}

// 低优先级：实际执行业务
class BusinessReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        // 只有通过了权限检查才能执行到这里
        doBusiness()
    }
}
```

### 场景 3：数据处理流水线

```kotlin
// 第 1 步：数据验证
class ValidationReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val data = intent.getStringExtra("data")
        if (data.isNullOrEmpty()) {
            abortBroadcast()  // 数据无效，截断
        } else {
            resultData = "验证通过"  // 给下一步标记
        }
    }
}

// 第 2 步：数据转换
class TransformReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (resultData == "验证通过") {
            // 转换数据
            val original = intent.getStringExtra("data")
            resultData = "转换后: ${original?.toUpperCase()}"
        }
    }
}

// 第 3 步：数据存储
class StorageReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        // 存储转换后的数据
        saveToDatabase(resultData)
    }
}
```

---

## 广播 vs Activity 传参

### 问题：广播是不是 Activity 的一种传参互动方式？

**答案：可以，但不推荐！**

### 为什么不推荐用广播做 Activity 传参？

| 方式 | 优点 | 缺点 |
|------|------|------|
| **Intent 直接传参** | 简单、类型安全、性能好 | 只能启动时传 |
| **广播** | 可以跨组件、解耦 | 开销大、异步、容易滥用 |

### 什么时候用广播？

✅ **推荐用广播**：
- 一对多通知（发送给多个接收者）
- 跨进程通信（不同 App 之间）
- 系统事件监听（网络、电量、开机等）
- App 未启动时也要收到（静态注册）

❌ **不推荐用广播**：
- Activity A → Activity B 传参（直接用 Intent）
- Fragment 和 Activity 通信（用接口回调或 ViewModel）
- 简单的回调通知（用接口或 LiveData）

### 正确示例对比

**错误：用广播传参** ❌

```kotlin
// Activity A
val intent = Intent("com.example.SEND_DATA")
intent.putExtra("username", "张三")
sendBroadcast(intent)
finish()  // 关闭自己

// Activity B
class DataReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val username = intent.getStringExtra("username")
        // 这样做太重了！
    }
}
```

**正确：用 Intent 传参** ✅

```kotlin
// Activity A
val intent = Intent(this, ActivityB::class.java)
intent.putExtra("username", "张三")
startActivity(intent)

// Activity B
val username = intent.getStringExtra("username")  // 简单直接！
```

**特殊情况：返回结果给上一个 Activity** ✅

```kotlin
// Activity A 启动 B 并等待结果
val intent = Intent(this, ActivityB::class.java)
startActivityForResult(intent, REQUEST_CODE)

// Activity B 返回结果
val resultIntent = Intent()
resultIntent.putExtra("result", "处理完成")
setResult(Activity.RESULT_OK, resultIntent)
finish()
```

---

## 常见误区澄清

### 误区 1：隐式跳转和静态广播是同一个机制

**❌ 错误理解**：
> "隐式跳转 Activity 也是用 intent-filter，和静态广播是一个东西"

**✅ 正确理解**：
- **隐式跳转**：让别的 App 能打开你的 **Activity**
  - 例子：浏览器点击链接打开你的 App
  - 接收者：`Activity`

- **静态广播**：让系统事件能唤醒你的 **BroadcastReceiver**
  - 例子：开机后启动服务
  - 接收者：`BroadcastReceiver`

**虽然都用 intent-filter，但接收者完全不同！**

---

### 误区 2：动态注册不注销也没关系

**❌ 错误**：
```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    registerReceiver(receiver, filter)
    // 忘记在 onDestroy 中 unregisterReceiver
}
```

**✅ 正确**：
```kotlin
override fun onDestroy() {
    super.onDestroy()
    unregisterReceiver(receiver)  // 必须注销！
}
```

**后果**：
- 内存泄漏（Activity 无法被回收）
- 崩溃（接收器在 Activity 销毁后收到广播）

---

### 误区 3：所有广播都需要权限

**❌ 错误**：
```kotlin
// 以为一定要加权限
sendOrderedBroadcast(intent, "com.example.PERMISSION")
```

**✅ 正确**：
```kotlin
// 不需要权限时传 null
sendBroadcast(intent)  // 无序广播，无需权限
sendOrderedBroadcast(intent, null)  // 有序广播，无需权限

// 需要权限时（如系统广播）
sendOrderedBroadcast(intent, "android.permission.RECEIVE_SMS")
```

---

### 误区 4：有序广播可以设置返回值给发送方

**❌ 错误理解**：
> "有序广播处理完可以返回值给发送方"

**✅ 正确理解**：
- `resultData` 是给**下一个接收器**看的，不是给发送方的
- 发送方无法直接获取接收器的处理结果
- 如果需要结果，应该用 `startActivityForResult` 或回调接口

---

## 总结

1. **无序广播**：最常用的广播，通知所有接收者
2. **有序广播**：需要优先级控制、数据修改、拦截时使用
3. **本地广播**：App 内部通信（已废弃，用 LiveData 替代）
4. **粘性广播**：历史消息会保留（已废弃，了解即可）
5. **动态注册**：临时监听，记得注销
6. **静态注册**：全局监听，App 未启动也能收到
7. **广播 ≠ Activity 传参方式**：传参用 Intent，广播用于通知和解耦

---

## 参考

- [Android 官方文档 - 广播](https://developer.android.com/guide/components/broadcasts)
- [LocalBroadcastManager 废弃说明](https://developer.android.com/reference/androidx/localbroadcastmanager/content/LocalBroadcastManager)
