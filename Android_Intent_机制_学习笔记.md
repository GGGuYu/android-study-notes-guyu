# Android Intent 机制学习笔记

## 一眼看懂

**Intent = 快递单** 📦

- **显式 Intent**：寄件人和收件人都清楚（知道具体要发给谁）
- **隐式 Intent**：只写要求和条件（系统帮找谁能处理）
- **作用**：四大组件（Activity/Service/Receiver/Provider）之间的通信桥梁

---

## 目录

1. [Intent 是什么](#intent-是什么)
2. [显式 vs 隐式 Intent](#显式-vs-隐式-intent)
3. [核心使用场景](#核心使用场景)
4. [传递数据的 N 种方式](#传递数据的-n-种方式)
5. [隐式 Intent 匹配规则](#隐式-intent-匹配规则)
6. [PendingIntent - 延迟执行](#pendingintent---延迟执行)
7. [常用系统 Intent](#常用系统-intent)
8. [避坑指南](#避坑指南)

---

## Intent 是什么

Intent（意图）是 Android 组件间通信的**标准方式**。

**三大用途**：
1. **启动 Activity**：页面跳转
2. **启动 Service**：后台任务
3. **发送广播**：全局通知

**基本结构**：
```kotlin
val intent = Intent(上下文, 目标组件::class.java)
intent.putExtra("key", value)  // 带数据
startActivity(intent)  // 执行
```

---

## 显式 vs 隐式 Intent

### 显式 Intent（明确指定）

**特点**：直接指定组件类名，效率高，用于 App **内部**跳转

```kotlin
// 启动 Activity
val intent = Intent(this, MainActivity::class.java)
startActivity(intent)

// 启动 Service
val serviceIntent = Intent(this, MusicService::class.java)
startService(serviceIntent)
```

**优势**：
- ✅ 性能好（直接找到组件，无需匹配）
- ✅ 类型安全（编译期检查）
- ✅ 不会跳错地方

### 隐式 Intent（模糊匹配）

**特点**：通过 **Action/Category/Data** 描述需求，系统帮你找能处理的组件

```kotlin
// 打开网页
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://google.com"))
startActivity(intent)

// 发送邮件
val emailIntent = Intent(Intent.ACTION_SENDTO).apply {
    data = Uri.parse("mailto:test@example.com")
    putExtra(Intent.EXTRA_SUBJECT, "主题")
}
startActivity(emailIntent)
```

**优势**：
- ✅ 解耦（不知道具体实现）
- ✅ 跨 App 调用（系统相机、浏览器等）
- ✅ 灵活（多个 App 可响应同一意图）

---

## 核心使用场景

### 1. Activity 跳转（带数据）

```kotlin
// Activity A 发送
val intent = Intent(this, DetailActivity::class.java).apply {
    putExtra("user_id", 123)
    putExtra("user_name", "张三")
    putExtra("is_vip", true)
}
startActivity(intent)

// Activity B 接收
val userId = intent.getIntExtra("user_id", 0)
val userName = intent.getStringExtra("user_name")
val isVip = intent.getBooleanExtra("is_vip", false)
```

### 2. 返回结果给上一个 Activity

```kotlin
// Activity A 启动并等待结果
val intent = Intent(this, SelectActivity::class.java)
startActivityForResult(intent, REQUEST_CODE)

// Activity B 返回结果
val resultIntent = Intent()
resultIntent.putExtra("selected_item", "选项1")
setResult(Activity.RESULT_OK, resultIntent)
finish()

// Activity A 接收结果（旧方式）
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)
    if (requestCode == REQUEST_CODE && resultCode == Activity.RESULT_OK) {
        val item = data?.getStringExtra("selected_item")
    }
}
```

**现代方式**（推荐）：
```kotlin
// 使用 Activity Result API
val launcher = registerForActivityResult(
    ActivityResultContracts.StartActivityForResult()
) { result ->
    if (result.resultCode == Activity.RESULT_OK) {
        val data = result.data?.getStringExtra("selected_item")
    }
}

launcher.launch(Intent(this, SelectActivity::class.java))
```

### 3. 启动 Service

```kotlin
// 方式 1：startService - 长期运行
val serviceIntent = Intent(this, DownloadService::class.java)
serviceIntent.putExtra("download_url", "https://example.com/file.zip")
startService(serviceIntent)

// 方式 2：bindService - 绑定交互
val bindIntent = Intent(this, MusicService::class.java)
bindService(bindIntent, serviceConnection, Context.BIND_AUTO_CREATE)
```

### 4. 发送广播

```kotlin
// 发送自定义广播
val broadcastIntent = Intent("com.example.DOWNLOAD_COMPLETE")
broadcastIntent.putExtra("file_path", "/downloads/song.mp3")
sendBroadcast(broadcastIntent)
```

---

## 传递数据的 N 种方式

### 方式 1：基础类型（最简单）

```kotlin
intent.putExtra("name", "张三")
intent.putExtra("age", 25)
intent.putExtra("score", 98.5f)
intent.putExtra("is_student", true)
```

### 方式 2：Bundle（批量打包）

```kotlin
val bundle = Bundle().apply {
    putString("key1", "value1")
    putInt("key2", 100)
    putBoolean("key3", true)
}
intent.putExtras(bundle)
```

### 方式 3：Serializable（Java 对象）

```kotlin
// 数据类实现 Serializable
@Serializable
data class User(
    val id: Int,
    val name: String
) : java.io.Serializable

// 传递
intent.putExtra("user", user)

// 接收
val user = intent.getSerializableExtra("user") as? User
```

**缺点**：性能差（需要反射，产生大量临时对象）

### 方式 4：Parcelable（推荐 ⭐）

```kotlin
// Kotlin 中使用 @Parcelize 注解（最简洁）
import kotlinx.parcelize.Parcelize

@Parcelize
data class Song(
    val id: String,
    val title: String,
    val artist: String,
    val duration: Long
) : Parcelable

// 传递
intent.putExtra("song", song)

// 接收
val song = intent.getParcelableExtra<Song>("song")
```

**优势**：性能好（手动序列化，无反射），是 Android 推荐方式

### 方式 5：Intent 传递限制 ⚠️

```kotlin
// ❌ 错误：传递大图会崩溃
val bitmap = getLargeBitmap()
intent.putExtra("image", bitmap)  // TransactionTooLargeException!

// ✅ 正确：传递路径
intent.putExtra("image_path", "/storage/image.jpg")

// ✅ 正确：传递 URI
intent.putExtra("image_uri", imageUri.toString())
```

**限制**：Intent 数据不能超过 **1MB**（Binder 缓冲区限制）

---

## 隐式 Intent 匹配规则

AndroidManifest.xml 中的 **intent-filter** 决定谁能响应：

```xml
<activity android:name=".BrowserActivity">
    <intent-filter>
        <!-- 必须匹配 action -->
        <action android:name="android.intent.action.VIEW" />
        
        <!-- 默认 category，系统会自动添加 -->
        <category android:name="android.intent.category.DEFAULT" />
        
        <!-- data 匹配（可选但常用）-->
        <data 
            android:scheme="https"
            android:host="www.example.com"
            android:pathPrefix="/music" />
    </intent-filter>
</activity>
```

### 匹配三要素

| 要素 | 作用 | 示例 |
|------|------|------|
| **Action** | 动作类型 | `ACTION_VIEW`、`ACTION_SEND`、`ACTION_EDIT` |
| **Category** | 附加类别 | `CATEGORY_DEFAULT`、`CATEGORY_BROWSABLE` |
| **Data** | 数据格式 | `https://`、`tel:`、`image/*` |

### 匹配示例

```kotlin
// 这个 Intent 会匹配上面的 filter
val intent = Intent(Intent.ACTION_VIEW).apply {
    data = Uri.parse("https://www.example.com/music/123")
}
startActivity(intent)
```

### 常见 Action 对照表

| Action | 用途 |
|--------|------|
| `ACTION_VIEW` | 查看内容（网页、图片等） |
| `ACTION_SEND` | 分享内容 |
| `ACTION_EDIT` | 编辑内容 |
| `ACTION_DIAL` | 拨打电话（不直接拨出） |
| `ACTION_CALL` | 直接拨打电话（需要权限） |
| `ACTION_MAIN` | 应用入口 |
| `ACTION_PICK` | 从列表中选择 |

---

## PendingIntent - 延迟执行

**场景**：通知栏点击、闹钟、桌面小部件

**特点**：不是立即执行，而是交给系统保管，在特定时刻触发

```kotlin
// 1. 创建普通 Intent
val intent = Intent(context, MusicActivity::class.java).apply {
    putExtra("action", "play")
    putExtra("song_id", songId)
}

// 2. 包装成 PendingIntent
val pendingIntent = PendingIntent.getActivity(
    context,
    0,  // requestCode
    intent,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)

// 3. 用于通知
val notification = NotificationCompat.Builder(context, CHANNEL_ID)
    .setSmallIcon(R.drawable.ic_play)
    .setContentTitle("正在播放")
    .setContentText(songTitle)
    .setContentIntent(pendingIntent)  // 点击时触发
    .build()

notificationManager.notify(NOTIFICATION_ID, notification)
```

**PendingIntent 的 4 种获取方式**：

| 方法 | 用途 |
|------|------|
| `getActivity()` | 启动 Activity |
| `getService()` | 启动 Service |
| `getBroadcast()` | 发送广播 |
| `getForegroundService()` | 启动前台 Service |

**Flags 说明**：
- `FLAG_UPDATE_CURRENT`：更新已有 PendingIntent 的 Extra 数据
- `FLAG_IMMUTABLE`：不可修改（Android 12+ 必需）
- `FLAG_CANCEL_CURRENT`：取消旧的，创建新的

---

## 常用系统 Intent

### 1. 打开浏览器

```kotlin
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://www.google.com"))
startActivity(intent)
```

### 2. 打开相机拍照

```kotlin
// 拍照并返回缩略图
val cameraIntent = Intent(MediaStore.ACTION_IMAGE_CAPTURE)
startActivityForResult(cameraIntent, REQUEST_CODE)

// 拍照并保存到指定路径
val photoUri = createImageUri()
val cameraIntent = Intent(MediaStore.ACTION_IMAGE_CAPTURE).apply {
    putExtra(MediaStore.EXTRA_OUTPUT, photoUri)
}
startActivityForResult(cameraIntent, REQUEST_CODE)
```

### 3. 打开相册选图

```kotlin
val galleryIntent = Intent(Intent.ACTION_PICK, MediaStore.Images.Media.EXTERNAL_CONTENT_URI)
startActivityForResult(galleryIntent, REQUEST_CODE_GALLERY)
```

### 4. 发送邮件

```kotlin
val emailIntent = Intent(Intent.ACTION_SENDTO).apply {
    data = Uri.parse("mailto:")
    putExtra(Intent.EXTRA_EMAIL, arrayOf("recipient@example.com"))
    putExtra(Intent.EXTRA_SUBJECT, "邮件主题")
    putExtra(Intent.EXTRA_TEXT, "邮件内容")
}
startActivity(Intent.createChooser(emailIntent, "选择邮件应用"))
```

### 5. 拨打电话

```kotlin
// 打开拨号界面（不需要权限）
val dialIntent = Intent(Intent.ACTION_DIAL, Uri.parse("tel:10086"))
startActivity(dialIntent)

// 直接拨打电话（需要 CALL_PHONE 权限）
val callIntent = Intent(Intent.ACTION_CALL, Uri.parse("tel:10086"))
startActivity(callIntent)
```

### 6. 发送短信

```kotlin
val smsIntent = Intent(Intent.ACTION_SENDTO).apply {
    data = Uri.parse("smsto:10086")
    putExtra("sms_body", "短信内容")
}
startActivity(smsIntent)
```

### 7. 打开系统设置

```kotlin
// 应用详情页
val settingsIntent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
    data = Uri.fromParts("package", packageName, null)
}
startActivity(settingsIntent)

// WiFi 设置
val wifiIntent = Intent(Settings.ACTION_WIFI_SETTINGS)
startActivity(wifiIntent)

// 定位设置
val locationIntent = Intent(Settings.ACTION_LOCATION_SOURCE_SETTINGS)
startActivity(locationIntent)
```

### 8. 分享内容

```kotlin
// 分享文本
val shareIntent = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_TEXT, "分享这段文字")
}
startActivity(Intent.createChooser(shareIntent, "分享到"))

// 分享图片
val shareImageIntent = Intent(Intent.ACTION_SEND).apply {
    type = "image/*"
    putExtra(Intent.EXTRA_STREAM, imageUri)
}
startActivity(Intent.createChooser(shareImageIntent, "分享图片"))
```

### 9. 打开地图

```kotlin
val mapIntent = Intent(Intent.ACTION_VIEW, Uri.parse("geo:37.7749,-122.4194"))
startActivity(mapIntent)
```

### 10. 安装 APK

```kotlin
val installIntent = Intent(Intent.ACTION_VIEW).apply {
    setDataAndType(apkUri, "application/vnd.android.package-archive")
    flags = Intent.FLAG_GRANT_READ_URI_PERMISSION
}
startActivity(installIntent)
```

---

## 避坑指南

### 坑 1：ActivityNotFoundException

**问题**：隐式 Intent 没有找到匹配的 Activity

```kotlin
// ❌ 错误：直接 startActivity 可能崩溃
startActivity(intent)  // ActivityNotFoundException!

// ✅ 正确：先检查是否能处理
if (intent.resolveActivity(packageManager) != null) {
    startActivity(intent)
} else {
    Toast.makeText(this, "没有找到合适的应用", Toast.LENGTH_SHORT).show()
}
```

### 坑 2：忘记在 onDestroy 注销

**问题**：动态注册的广播接收器不注销导致内存泄漏

```kotlin
override fun onDestroy() {
    super.onDestroy()
    unregisterReceiver(receiver)  // 必须注销！
}
```

### 坑 3：Intent 数据过大

**问题**：传递大图或大对象导致崩溃

```kotlin
// ❌ 错误
intent.putExtra("bitmap", largeBitmap)

// ✅ 正确：传递路径或 URI
intent.putExtra("image_path", imagePath)
```

### 坑 4：Android 12+ exported 属性

**问题**：Android 12 后 Activity 必须显式声明 exported

```xml
<!-- ✅ 正确 -->
<activity 
    android:name=".MainActivity"
    android:exported="true">  <!-- 能否被其他 App 启动 -->
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

### 坑 5：隐式 Intent 启动 Service（Android 8.0+ 限制）

**问题**：Android 8.0 后禁止后台应用通过隐式 Intent 启动 Service

```kotlin
// ❌ Android 8.0+ 会报错
val serviceIntent = Intent("com.example.ACTION_SERVICE")
startService(serviceIntent)

// ✅ 正确：使用显式 Intent
val serviceIntent = Intent(this, MyService::class.java)
startService(serviceIntent)
```

### 坑 6：PendingIntent 重复问题

**问题**：多个 PendingIntent 使用相同的 requestCode，后面的覆盖前面的

```kotlin
// ❌ 错误：所有通知点击都跳转到同一个页面
val pendingIntent = PendingIntent.getActivity(context, 0, intent, flags)

// ✅ 正确：使用不同的 requestCode
val pendingIntent1 = PendingIntent.getActivity(context, 1, intent1, flags)
val pendingIntent2 = PendingIntent.getActivity(context, 2, intent2, flags)
```

---

## 总结

| 概念 | 一句话总结 |
|------|-----------|
| **Intent** | 组件间通信的快递单 |
| **显式 Intent** | 直接指定收件人（App 内部） |
| **隐式 Intent** | 描述需求，系统匹配（跨 App） |
| **Action** | 要做什么（VIEW、SEND、EDIT） |
| **Category** | 附加条件（DEFAULT、BROWSABLE） |
| **Data** | 数据格式（https://、tel:） |
| **Bundle** | 装数据的包裹 |
| **Parcelable** | 高性能对象传输方式 |
| **PendingIntent** | 延迟执行的定时炸弹 |
| **Intent Filter** | 谁能处理这个意图的筛选条件 |

**核心口诀**：
- 内部跳转用**显式**，外部调用用**隐式**
- 传大数据用**路径**，传对象用**Parcelable**
- 延迟执行用**PendingIntent**，记得加 **FLAG_IMMUTABLE**
- 启动之前先**检查**，避免 **ActivityNotFoundException**

---

## 参考

- [Android 官方文档 - Intent 和 Intent Filter](https://developer.android.com/guide/components/intents-filters)
- [Activity Result API](https://developer.android.com/training/basics/intents/result)
- [PendingIntent 安全最佳实践](https://developer.android.com/guide/components/intents#pending-intent-security)
