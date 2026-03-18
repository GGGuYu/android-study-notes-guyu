# Android Handler 机制学习笔记

## 我的理解

我们可以有很多的 Handler，比如在一个 Activity 中，可以创建一个或多个 Handler。然后需要给 Handler 绑定 Looper，一般就是绑定**主线程 Looper**，因为这样的话可以更新 UI。

当我需要做一个异步操作时，比如开启一个子线程做耗时操作，做完之后就可以调用 Handler 去发送消息。可以发一个 Runnable 任务，目的是为了通知主线程更新 UI。

Looper 找到这个消息之后：
- 如果这个消息有 callback（也就是 Runnable），就直接运行
- 如果没有 callback，就交给 Handler 调用重写的 `handleMessage()` 方法处理

## 重要纠正与补充

### 1. Handler 与 Looper 的绑定
Handler 在**创建时**就绑定了 Looper，不是创建后绑定的：

```kotlin
// 方式1：显式指定主线程 Looper
val handler = Handler(Looper.getMainLooper())

// 方式2：默认绑定当前线程的 Looper
val handler2 = Handler()
```

### 2. 线程与 Looper 的关系
- **一个线程只能有一个 Looper**（通过 ThreadLocal 保证）
- **一个 Looper 对应一个 MessageQueue**
- **一个 Looper 可以有多个 Handler**

```
线程 → Looper → MessageQueue（一对一）
        ↓
     Handler（多对一）
```

### 3. Runnable 的本质
```kotlin
// post(Runnable) 内部会转成 Message
val msg = Message.obtain()
msg.callback = runnable  // Runnable 就是 Message 的 callback
```

### 4. 完整的消息处理流程
```
发送消息 → MessageQueue → Looper.loop() → Handler.dispatchMessage()
                                              ↓
                                  msg.callback != null ?
                                        ↓
                                是 → 执行 Runnable.callback
                                否 → Handler.handleMessage()
```

## 子线程创建 Handler 的正确方式

```kotlin
Thread {
    Looper.prepare()  // 1. 创建 Looper 和 MessageQueue
    val handler = Handler(Looper.myLooper()!!)
    
    // 使用 handler...
    
    Looper.loop()     // 2. 开始循环（会阻塞当前线程）
}.start()
```

## 内存泄漏注意事项

**错误做法**（匿名内部类持有外部 Activity 引用）：
```kotlin
val handler = object : Handler(Looper.getMainLooper()) {
    override fun handleMessage(msg: Message) {
        // 如果 Activity 已销毁，会导致泄漏！
    }
}
```

**正确做法**（使用静态内部类 + WeakReference）：
```kotlin
class MyHandler(activity: Activity) : Handler(Looper.getMainLooper()) {
    private val weakRef = WeakReference(activity)
    
    override fun handleMessage(msg: Message) {
        val activity = weakRef.get() ?: return
        // 安全地使用 activity
    }
}
```

## 性能优化：Message 复用

```kotlin
// ❌ 不推荐：每次都创建新的 Message
val msg = Message()

// ✅ 推荐：使用对象池复用
val msg = Message.obtain()  // 从对象池获取
```

## 一句话总结

> Handler 是**消息的发送者和处理者**，Looper 是**消息的调度员**，MessageQueue 是**消息的队列**，它们三者配合实现了 Android 的线程间通信机制。

---

记录日期：2026年3月18日
学习内容：Handler、Looper、Message、MessageQueue 的关系与工作流程
