# Android ANR (Application Not Responding) 学习笔记

## 什么是 ANR

ANR 是 **Application Not Responding** 的缩写，中文意思是"应用程序无响应"。

当 Android 应用程序在主线程（UI 线程）上执行耗时操作，导致无法在系统规定的时间内响应用户输入或其他系统事件时，系统就会弹出 ANR 对话框，提示用户应用无响应。

---

## ANR 的触发条件

| 场景 | 超时时间 | 说明 |
|------|---------|------|
| **输入事件**（触摸、按键） | 5秒 | 用户点击、滑动后，主线程5秒内未处理完成 |
| **前台广播接收器** | 10秒 | 前台应用的 BroadcastReceiver 处理超时 |
| **后台广播接收器** | 60秒 | 后台应用的 BroadcastReceiver 处理超时 |
| **前台服务启动** | 5秒 | 调用 `startForegroundService()` 后，必须在5秒内调用 `startForeground()` |
| **ContentProvider** | 10秒 | ContentProvider 查询超时 |

⚠️ **常见误解纠正**：
- 前台服务是 **5秒**，不是20秒
- 前台广播是 **10秒**，后台广播才是60秒

---

## ANR 的根本原因

### 1. 主线程执行耗时操作
- 网络请求（HTTP 请求）
- 数据库查询（大量数据查询）
- 文件读写操作
- 复杂计算（图像处理、大量数据排序）
- 死锁或锁竞争

### 2. 关键系统组件
- **ActivityManagerService (AMS)**：负责监控应用响应状态，超时后触发 ANR
- **Choreographer**：负责监控帧率，虽然不会直接触发 ANR，但可以检测到卡顿（掉帧）

---

## 如何排查 ANR

### 方法一：查看 trace.txt 日志

ANR 发生时，系统会在 `/data/anr/traces.txt` 中保存堆栈信息。

**查看重点**：
1. **"main" 线程状态**
   - `RUNNABLE`：正在运行
   - `BLOCKED`：被锁阻塞 ⚠️
   - `WAITING` / `TIMED_WAITING`：等待锁或条件 ⚠️

2. **堆栈顶部方法**
   ```
   at com.example.app.MainActivity.onClick(MainActivity.java:45)
   at android.view.View.performClick(View.java:7125)
   ```

3. **锁竞争信息**
   ```
   waiting to lock <0x12345678> (a java.lang.Object)
   held by thread 15
   ```

4. **CPU 使用情况**
   - 查看是 CPU 繁忙还是 IO 等待

### 方法二：使用 Systrace

Systrace 可以可视化查看系统调用和 CPU 使用情况：

```bash
# 使用 Android Studio 的 Profiler 或命令行
python systrace.py -o trace.html sched gfx view wm am app
```

**看什么**：
- 主线程是否有长任务
- CPU 是否满载
- 是否有锁等待

### 方法三：使用 Android Studio Profiler

- **CPU Profiler**：查看方法耗时
- **Memory Profiler**：排查内存泄漏导致的卡顿

---

## 如何避免 ANR

### 1. 不要在主线程执行耗时操作

```kotlin
// ❌ 错误示例：在主线程进行网络请求
override fun onClick(v: View) {
    val response = httpClient.get("https://api.example.com") // 会导致 ANR！
    textView.text = response
}

// ✅ 正确示例：使用协程
override fun onClick(v: View) {
    lifecycleScope.launch {
        val response = withContext(Dispatchers.IO) {
            httpClient.get("https://api.example.com")
        }
        textView.text = response
    }
}
```

### 2. 使用异步处理

- **协程 (Kotlin Coroutines)**：推荐
- **RxJava**：响应式编程
- **AsyncTask**（已废弃）：使用 WorkManager 替代
- **WorkManager**：后台任务
- **HandlerThread / ThreadPool**：多线程处理

### 3. 避免死锁

```kotlin
// ❌ 错误：嵌套锁可能导致死锁
fun methodA() {
    synchronized(lockA) {
        synchronized(lockB) { // 如果另一个线程反向获取锁，会死锁
            // do something
        }
    }
}

// ✅ 正确：始终按相同顺序获取锁
fun methodA() {
    val first = if (lockA.hashCode() < lockB.hashCode()) lockA else lockB
    val second = if (lockA.hashCode() < lockB.hashCode()) lockB else lockA
    synchronized(first) {
        synchronized(second) {
            // do something
        }
    }
}
```

### 4. 优化启动时间

- **Application.onCreate()** 中不要做耗时操作
- 使用**懒加载**初始化组件
- 考虑使用 **App Startup** 库优化启动流程

---

## 总结

| 检查项 | 建议 |
|--------|------|
| 网络请求 | 使用协程 + IO 线程 |
| 数据库操作 | 使用 Room + 协程/Flow |
| 文件读写 | 使用后台线程 |
| 复杂计算 | 使用协程或线程池 |
| 锁竞争 | 减少锁粒度，避免死锁 |
| 启动优化 | Application 中不要初始化过多组件 |

---

## 参考资源

- [Android 官方文档 - ANR](https://developer.android.com/topic/performance/vitals/anr)
- [Android 性能优化最佳实践](https://developer.android.com/topic/performance)

---

*学习日期：2026年3月25日*
