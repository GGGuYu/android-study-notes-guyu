# 音频焦点 (Audio Focus) - 助眠APP项目难点

## 面试场景话术

### 问题背景

> 我在开发助眠APP时，遇到一个很影响用户体验的问题：用户睡前播放白噪音或轻音乐助眠，但如果在睡前刷短视频、接电话或有消息通知时，助眠音频不会自动暂停，而是和其他声音混叠在一起，非常刺耳，破坏了助眠氛围。

**具体痛点**：
1. **来电时**：铃声和助眠音频同时播放，用户被双重声音吵醒
2. **刷视频时**：视频声音和助眠音频重叠，体验很差
3. **切换音乐APP时**：两个播放器同时工作，电池消耗快
4. **通知提醒**：系统提示音打断助眠节奏

---

## 解决方案

### 方案对比

| 方案 | 实现方式 | 优点 | 缺点 |
|------|----------|------|------|
| **手动管理** | 使用 `AudioManager.requestAudioFocus()` | 控制粒度细 | 代码复杂，需处理所有焦点变化场景 |
| **ExoPlayer自动** | `setAudioAttributes(audioAttributes, true)` | 自动处理，代码简洁 | 封装度高，灵活性略低 |

### 选择的方案

采用 **Media3 ExoPlayer 内置的音频焦点管理**：

```kotlin
// MediaService.kt
private val player: Player by lazy {
    ExoPlayer.Builder(this).build().apply {
        // 启用音频焦点自动管理
        setAudioAttributes(audioAttributes, /* handleAudioFocus= */ true)
        // 处理音频输出变化（如拔出耳机）
        setHandleAudioBecomingNoisy(true)
        addListener(playerEventListener)
    }
}
```

---

## 音频焦点工作机制

### 焦点类型

Android 系统提供了几种音频焦点请求类型：

```kotlin
// AudioManager 中的焦点类型
AUDIOFOCUS_GAIN          // 长时间持有焦点（如音乐播放）
AUDIOFOCUS_GAIN_TRANSIENT      // 短暂持有（如导航提示）
AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK  // 短暂持有，可duck其他应用（如通知）
```

### 焦点变化回调

当其他应用请求焦点时，系统会回调 `OnAudioFocusChangeListener`：

```kotlin
interface OnAudioFocusChangeListener {
    fun onAudioFocusChange(focusChange: Int)
}

// 可能的 focusChange 值：
AUDIOFOCUS_GAIN          // 获得焦点 → 恢复播放
AUDIOFOCUS_LOSS          // 永久失去 → 暂停，不再自动恢复
AUDIOFOCUS_LOSS_TRANSIENT     // 短暂失去 → 暂停，稍后可能恢复
AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK  // 短暂失去，可降低音量继续播放
```

---

## 流程图：音频焦点处理流程

```
┌─────────────────────────────────────────────────────────────┐
│                     助眠APP开始播放                          │
│                MediaService.onCreate()                       │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              ExoPlayer 请求音频焦点                          │
│         AudioFocusManager.requestAudioFocus()                │
│              ↓                                               │
│    AudioManager.requestAudioFocus()                          │
└───────────────────────┬─────────────────────────────────────┘
                        │
            ┌───────────┴───────────┐
            │                       │
            ▼                       ▼
┌──────────────────┐     ┌──────────────────┐
│   获得焦点 ✅     │     │   被拒绝 ❌       │
│   开始播放音频    │     │   等待其他应用释放  │
└────────┬─────────┘     └──────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│                   正常运行中...                              │
└───────────────────────┬─────────────────────────────────────┘
                        │
         ┌──────────────┼──────────────┐
         │              │              │
         ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   来电 📞     │ │   刷视频 🎬   │ │   通知 📢     │
│              │ │              │ │              │
│ 其他应用请求  │ │ 其他应用请求  │ │ 其他应用请求  │
│   焦点       │ │   焦点       │ │   焦点       │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │                │                │
       └────────────────┴────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              ExoPlayer 收到焦点变化回调                        │
│         onAudioFocusChange(focusChange)                      │
└───────────────────────┬─────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│LOSS_TRANSIENT│  │LOSS_TRANSIENT│  │   LOSS      │
│             │  │   CAN_DUCK   │  │  (永久失去)  │
│             │  │              │  │             │
│   暂停播放   │  │  降低音量    │  │   暂停播放   │
│  等待恢复    │  │  继续播放    │  │  不自动恢复  │
└──────┬──────┘  └──────┬──────┘  └─────────────┘
       │                │
       │                │
       ▼                ▼
┌─────────────────────────────────────────────────────────────┐
│           来电结束 / 视频关闭 / 通知播放完毕                  │
│              其他应用释放焦点                                │
│                   ↓                                         │
│          收到 AUDIOFOCUS_GAIN                               │
│                   ↓                                         │
│              恢复播放助眠音频                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 扩展优化：耳机拔出自动暂停

```kotlin
// 启用 "音频变吵" 处理
setHandleAudioBecomingNoisy(true)
```

**场景**：用户戴着耳机听白噪音入睡，不小心拔出耳机
**效果**：音频自动暂停，避免突然外放吵醒室友或家人

**原理**：系统发送 `ACTION_AUDIO_BECOMING_NOISY` 广播，ExoPlayer 自动监听并暂停

---

## 简历写法

```
项目难点：
- 音频焦点管理：解决助眠音频与其他应用音频冲突问题，实现来电/通知/切换应用时的智能暂停和恢复
- 技术方案：利用 Media3 ExoPlayer 内置音频焦点机制，避免手动管理复杂度，确保低功耗和流畅体验
- 优化细节：增加耳机拔出自动暂停功能，提升用户体验
```

---

## 面试官可能追问

### Q1: 如果用户不想自动暂停怎么办？

**回答思路**：提供用户设置选项

```kotlin
// 设置界面提供选项
enum class InterruptionBehavior {
    PAUSE,           // 暂停（默认）
    DUCK,            // 降低音量继续播放
    CONTINUE         // 继续播放
}

// 根据用户选择动态设置音频属性
val audioAttributes = AudioAttributes.Builder()
    .setUsage(C.USAGE_MEDIA)
    .setContentType(C.AUDIO_CONTENT_TYPE_MUSIC)
    .build()

// handleAudioFocus 根据设置决定是否启用
player.setAudioAttributes(audioAttributes, handleAudioFocus)
```

### Q2: ExoPlayer 音频焦点内部原理是什么？

**回答要点**：
1. ExoPlayer 内部使用 `AudioFocusManager` 类管理焦点
2. `AudioFocusManager` 调用系统 `AudioManager.requestAudioFocus()`
3. 注册 `OnAudioFocusChangeListener` 监听焦点变化
4. 根据回调自动调用 `player.play()` 或 `player.pause()`

```kotlin
// 简化的伪代码
class AudioFocusManager {
    fun requestAudioFocus() {
        audioManager.requestAudioFocus(
            focusListener,
            AudioManager.STREAM_MUSIC,
            AudioManager.AUDIOFOCUS_GAIN
        )
    }
    
    private val focusListener = OnAudioFocusChangeListener { focusChange ->
        when (focusChange) {
            AUDIOFOCUS_GAIN -> player.play()
            AUDIOFOCUS_LOSS, 
            AUDIOFOCUS_LOSS_TRANSIENT -> player.pause()
        }
    }
}
```

### Q3: 与其他音频应用（如网易云、QQ音乐）的兼容性如何？

**回答**：
- 所有应用都遵循 Android 音频焦点规范
- 当用户打开其他音乐APP时，系统会回调给我们的应用，我们自动暂停
- 当用户关闭其他音乐APP时，系统归还焦点，我们自动恢复播放
- 这是系统级协调，不需要应用间直接通信

---

## 关键点总结

1. **助眠场景特殊性**：用户期望安静环境，不能有多重声音干扰
2. **音频焦点是必选项**：任何播放音频的APP都应该正确处理焦点
3. **ExoPlayer自动管理**：简化开发，避免重复造轮子
4. **耳机拔出处理**：细节体验，体现产品思维
5. **用户可控**：提供设置选项，满足不同用户需求

---

**创建时间**: 2025-03-18  
**标签**: Media3, ExoPlayer, AudioFocus, 面试技巧, 项目难点

**关联笔记**: [音视频播放流程对比](音视频播放流程对比.md)
