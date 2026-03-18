# Activity 学习笔记

> 基于实际项目 ClientApp 的学习总结  
> 日期：2026-03-18

## 一、Activity 是什么？

想象 Activity 是一个**页面容器**，用户看到的每个界面（登录页、首页、设置页）都是一个 Activity。  
类比：就像一本书的每一页，Activity 就是安卓应用的"页面"。

---

## 二、生命周期详解

### 2.1 生命周期回调方法

```
onCreate()    → 页面创建（初始化 UI、数据）
     ↓
onStart()     → 页面可见（但还不能交互）
     ↓
onResume()    → 页面可交互（用户能点击、滑动）
     ↓
onPause()     → 页面失去焦点（被其他页面覆盖）
     ↓
onStop()      → 页面不可见（但在内存中）
     ↓
onDestroy()   → 页面销毁（释放资源）
```

**记忆口诀：**
- **Create/Start/Resume**：创建 → 可见 → 可交互（正向流程）
- **Pause/Stop/Destroy**：暂停 → 隐藏 → 销毁（反向流程）

---

### 2.2 场景一：A 跳转到 B（正常跳转）

**流程图：**

```
时间点    A页面                    B页面
─────────────────────────────────────────────
0        onCreate()               
         ↓                        
1        onStart()                
         ↓                        
2        onResume()  ← 用户看到A   
         ↓                        
3        [用户点击跳转]             
         ↓                        
4        onPause()                
                                  onCreate()
                                  ↓
5                                 onStart()
                                  ↓
6                                 onResume()  ← 用户看到B
         ↓                        
7        onStop()   ← A还在内存里   
```

**关键理解：**
- A 先 `onPause()`，B 才能 `onResume()`（**焦点转移顺序**）
- A 没有销毁，只是 `onStop()`，所以**返回时很快**
- 性能优化：A 保持在内存，用户返回时秒开

**实际项目中的应用（ClientApp）：**

```kotlin
// MainActivity.kt:300-314
override fun onResume() {
    super.onResume()
    // 应用回到前台，隐藏桌面歌词
    globalLyricManager.tryHide()
}

override fun onPause() {
    super.onPause()
    // 应用进入后台，显示桌面歌词
    globalLyricManager.tryShow()
}
```

---

### 2.3 场景二：B 返回到 A

**流程图：**

```
时间点    B页面                    A页面
─────────────────────────────────────────────
0                                 (A在onStop状态)
         ↓                        
1        [用户点击返回]             
         ↓                        
2        onPause()                
                                  onRestart()  ← 不用重新创建！
                                  ↓
3                                 onStart()
                                  ↓
4                                 onResume()  ← 用户看到A
         ↓                        
5        onStop()                 
         ↓                        
6        onDestroy()  ← B被销毁    
```

**关键理解：**
- A 没有被销毁，所以走 `onRestart()` 而不是 `onCreate()`
- **onRestart() 只会在从 stop 状态恢复时调用**
- B 被彻底销毁（因为 finish() 了）

---

### 2.4 场景三：弹出 Dialog（依附于当前 Activity）

**Dialog 是 Activity 的一部分**，不影响生命周期：

```
A: onResume() → [弹出 Dialog] → A: onResume() (不变！)
                [关闭 Dialog] → A: onResume() (不变！)
```

**特点：**
- Dialog 只是**覆盖层**，Activity 仍是焦点持有者
- 生命周期**完全不变**

---

### 2.5 场景四：透明 DialogActivity（特殊的 Activity）

某些 Dialog 其实是**透明的 Activity**（主题设置为透明）：

**流程图：**

```
时间点    A页面                    Dialog
─────────────────────────────────────────────
0        onResume()  ← 用户操作A   
         ↓                        
1        [弹出透明Dialog]          
         ↓                        
2        onPause()   ← A仍可见！   
                                  onCreate()
                                  ↓
3                                 onStart()
                                  ↓
4                                 onResume()  ← 用户操作Dialog
```

**为什么 A 是 onPause 而不是 onStop？**
- 因为 A **仍然可见**（Dialog 是透明的）
- onPause = 失去焦点但仍可见
- onStop = 完全不可见

**关闭 Dialog 时：**

```
Dialog: onPause()
A: onResume()      ← 直接恢复，不需要 onRestart！
Dialog: onStop() → onDestroy()
```

**关键区别：**
- 如果 A 是 `onStop` → 恢复时要走 `onRestart()` → `onStart()` → `onResume()`
- 如果 A 是 `onPause` → 恢复时直接 `onResume()`

---

## 三、启动模式（LaunchMode）

### 3.1 四种模式对比

| 模式 | 特点 | 使用场景 |
|------|------|----------|
| **standard** | 每次启动都创建新实例 | 默认模式，大部分页面 |
| **singleTop** | 栈顶复用，否则创建 | 接收通知跳转的页面 |
| **singleTask** | 栈内唯一，复用时清上面的页面 | 主页、主界面 |
| **singleInstance** | 独占一个任务栈 | 极少使用 |

### 3.2 记忆技巧：想象一叠盘子 🥘

```
standard：        每次加个新盘子
singleTop：       最上面是同款就用，不是就加新的
singleTask：      整个厨房只能有一个这种盘子，
                  用它时把上面的都拿走
singleInstance：  这个盘子单独放一个架子，架子上只有它
```

### 3.3 项目实战：为什么 MainActivity 用 singleTask？

```xml
<!-- AndroidManifest.xml:60-69 -->
<activity
    android:name=".MainActivity"
    android:launchMode="singleTask">
```

**原因：**

1. **防止重复创建**：用户从通知栏、桌面歌词返回时，不新建实例
2. **统一任务栈**：微信/QQ 登录回调、外部链接打开都回到同一个 MainActivity
3. **处理外部 Intent**：通过 `onNewIntent()` 接收新请求

```kotlin
// MainActivity.kt:176-179
override fun onNewIntent(intent: Intent?) {
    super.onNewIntent(intent)
    handleIntent(intent)  // 处理外部跳转请求
}
```

---

## 四、页面跳转传参

### 4.1 A → B 正向传参

```kotlin
// 方式1：直接传基本类型
val intent = Intent(this, ActivityB::class.java)
intent.putExtra("userId", 123)
intent.putExtra("userName", "张三")
startActivity(intent)

// 方式2：Bundle 打包多个值
val bundle = Bundle().apply {
    putString("name", "张三")
    putInt("age", 20)
}
intent.putExtras(bundle)

// 方式3：传对象（推荐 Parcelable）
val user = User("张三", 20)
intent.putExtra("user", user)
```

### 4.2 B → A 返回传参

**现代方式（Activity Result API）：**

```kotlin
// A 页面注册回调
private val launcher = registerForActivityResult(
    ActivityResultContracts.StartActivityForResult()
) { result ->
    if (result.resultCode == RESULT_OK) {
        val data = result.data?.getStringExtra("result_key")
        // 处理返回数据
    }
}

// 启动 B
launcher.launch(Intent(this, ActivityB::class.java))

// B 页面返回数据
val intent = Intent().apply {
    putExtra("result_key", "返回的数据")
}
setResult(RESULT_OK, intent)
finish()
```

**项目实战（权限申请）：**

```kotlin
// MainActivity.kt:277-298
private val requestDrawOverlaysCallback: ActivityResultLauncher<Intent> =
    registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->
        if (Settings.canDrawOverlays(this)) {
            // 授权成功，显示桌面歌词
            viewModel.enableGlobalLyric()
        }
    }
```

**为什么用返回传参？**
- 系统权限（如悬浮窗）必须跳转到系统设置页
- 需要知道用户是否授权，再决定是否开启功能
- 常见的还有：选择图片、文件、蓝牙/WiFi 设置

---

## 五、重要知识点总结

### 5.1 生命周期状态对照

```
状态        | 可见？ | 可交互？ | 回调
────────────┼───────┼─────────┼──────────────────
Created     | 否    | 否      | onCreate()
Started     | 是    | 否      | onStart()
Resumed     | 是    | 是      | onResume()
Paused      | 是/否 | 否      | onPause()
Stopped     | 否    | 否      | onStop()
Destroyed   | 否    | 否      | onDestroy()
```

### 5.2 常见问题

**Q1：onPause 和 onStop 的区别？**
- onPause：失去焦点，但仍可能可见（如透明 Dialog）
- onStop：完全不可见

**Q2：为什么从 B 返回 A，A 不重新创建？**
- 因为 A 还在**任务栈**中，只是被 stop 了
- 返回时走 onRestart，不是 onCreate

**Q3：Dialog 和 DialogActivity 的区别？**
- Dialog：生命周期无影响
- DialogActivity：A 会 onPause（透明）或 onStop（不透明）

**Q4：singleTask 和 singleInstance 的区别？**
- singleTask：可以和其他 Activity 共存一个栈
- singleInstance：独占一个任务栈，整个栈只有它自己

---

## 六、参考资料

- [Android 官方文档 - Activity 生命周期](https://developer.android.com/guide/components/activities/activity-lifecycle)
- 项目源码：`MainActivity.kt`、`AndroidManifest.xml`

---

## 七、学习心得

**自己的话总结：**

> Activity 的生命周期就像是页面的"生命状态"，从创建到销毁有一系列回调方法。理解这些回调的时机很重要，比如 `onResume` 是可交互状态，`onPause` 是失去焦点。当 A 跳转到 B 时，A 先 pause，B 再 resume，最后 A 才 stop，这样 A 保持在内存里，返回时很快。启动模式用来控制 Activity 的创建和复用，MainActivity 用 singleTask 是为了防止重复创建，处理外部跳转。传参分正向和返回两种，返回传参常用于权限申请等场景。

**易混淆点：**
- onPause vs onStop（可见性区别）
- onRestart 只在从 stop 恢复时调用
- singleTask 清栈顶，singleInstance 独占栈

---

*最后更新：2026-03-18*