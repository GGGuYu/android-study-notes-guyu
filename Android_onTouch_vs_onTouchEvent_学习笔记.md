# Android onTouch vs onTouchEvent 学习笔记

## 一句话总结

- **onTouch**（通过 `setOnTouchListener`）：请个临时保姆，简单方便但能力有限
- **onTouchEvent**（重写 View 方法）：自己当家长，复杂但灵活，想怎么管就怎么管

---

## 对比表

| 特性 | onTouch (OnTouchListener) | onTouchEvent (View 方法) |
|------|---------------------------|-------------------------|
| **身份** | 外部监听者（接口实现） | View 自己的方法（继承重写） |
| **写法** | `view.setOnTouchListener { v, event -> ... }` | `override fun onTouchEvent(event: MotionEvent)` |
| **访问内部状态** | ❌ 只能看到 public 属性 | ✅ 可以访问所有内部变量和方法 |
| **修改绘制** | ❌ 无法直接触发重绘 | ✅ 可以调用 `invalidate()` |
| **事件拦截** | ❌ 无法实现 | ✅ 可配合 `onInterceptTouchEvent` |
| **复杂手势** | ⚠️ 困难（如拖拽） | ✅ 容易实现 |
| **适用场景** | 简单按钮反馈 | 自定义 View、拖拽、复杂交互 |

---

## 详细解释

### 1. 什么是"接口的一部分"？

**接口 = 合同**，规定了要实现哪些功能：

```kotlin
// Android 源码中的接口定义
interface OnTouchListener {
    fun onTouch(v: View, event: MotionEvent): Boolean
}
```

当你写：

```kotlin
button.setOnTouchListener { view, event ->
    // 这个 Lambda 就是在实现 onTouch 方法
    true
}
```

**本质就是**：在实现 `OnTouchListener` 接口，重写它的 `onTouch` 方法。

完整写法 vs Kotlin 简写：

```kotlin
// 完整写法（显式实现接口）
button.setOnTouchListener(object : View.OnTouchListener {
    override fun onTouch(v: View, event: MotionEvent): Boolean {
        Log.d("Touch", "坐标: ${event.x}, ${event.y}")
        return true
    }
})

// Kotlin SAM 转换简写
button.setOnTouchListener { v, event ->
    Log.d("Touch", "坐标: ${event.x}, ${event.y}")
    true
}
```

---

### 2. 调用顺序（重要！）

```
用户触摸屏幕
    ↓
1. 先检查是否有 OnTouchListener（setOnTouchListener 设置的）
    ↓
    ├─ 如果有，调用 onTouch()
    │   ├─ 返回 true → 事件结束，onTouchEvent **不会**被调用
    │   └─ 返回 false → 继续传递给 onTouchEvent
    │
    └─ 如果没有，直接进入下一步
    ↓
2. 调用 onTouchEvent()（View 自己的处理方法）
```

**关键区别**：
- `onTouch` 返回 `true` = 我全包了，别找别人
- `onTouch` 返回 `false` = 我处理不了，交给 onTouchEvent

---

## 代码示例

### 示例 1：onTouch（简单按钮效果）

适合场景：给现有按钮加点视觉反馈

```kotlin
val button = findViewById<Button>(R.id.myButton)

button.setOnTouchListener { v, event ->
    when (event.action) {
        MotionEvent.ACTION_DOWN -> {
            // 手指按下时变暗
            v.alpha = 0.5f
            Log.d("Touch", "按下了")
        }
        MotionEvent.ACTION_MOVE -> {
            // 手指移动
            Log.d("Touch", "移动中: ${event.x}, ${event.y}")
        }
        MotionEvent.ACTION_UP -> {
            // 手指抬起时恢复
            v.alpha = 1.0f
            Log.d("Touch", "抬起了")
        }
    }
    // false: 继续传递给 onTouchEvent，Button 能正常响应点击
    // true: 拦截事件，Button 的点击事件会失效
    false
}
```

**优点**：
- ✅ 简单，一行代码搞定
- ✅ 不用创建新类
- ✅ 可以随时添加、移除

**缺点**：
- ❌ 无法直接修改 View 内部状态
- ❌ 难以实现复杂交互（如拖拽）

---

### 示例 2：onTouchEvent（自定义 View）

适合场景：创建可拖拽的自定义 View、悬浮窗等

**项目实际案例**（悬浮歌词组件）：

```kotlin
class GlobalLyricView @JvmOverloads constructor(
    context: Context, 
    attrs: AttributeSet? = null
) : FrameLayout(context, attrs) {

    private var lastX = 0f
    private var lastY = 0f
    private val touchSlop = ViewConfiguration.get(context).scaledTouchSlop
    
    var onGlobalLyricDrag: ((Int) -> Unit)? = null

    override fun onTouchEvent(event: MotionEvent): Boolean {
        when (event.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
                lastX = event.x
                lastY = event.y
            }
            
            MotionEvent.ACTION_MOVE -> {
                // 计算滑动距离
                val distanceX = event.x - lastX
                val distanceY = event.y - lastY
                
                // 判断是否超过触发阈值
                if (abs(distanceY.toDouble()) > touchSlop) {
                    // 获取绝对坐标（包含状态栏）
                    val rawY = event.rawY
                    val moveY = rawY - lastY
                    
                    // 回调给外部，实际移动悬浮窗位置
                    onGlobalLyricDrag?.invoke(moveY.toInt())
                }
            }
        }
        return super.onTouchEvent(event)
    }
}
```

**优点**：
- ✅ 可以访问所有内部属性和方法
- ✅ 可以调用 `invalidate()` 触发重绘
- ✅ 可以配合 `onInterceptTouchEvent` 实现事件拦截
- ✅ 适合复杂手势（拖拽、缩放、旋转等）

**缺点**：
- ❌ 需要创建自定义 View 类
- ❌ 代码量相对较多

---

### 示例 3：完整的手势处理（拖拽 View）

```kotlin
class DraggableView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : View(context, attrs) {

    private var lastX = 0f
    private var lastY = 0f
    private var isDragging = false

    override fun onTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> {
                lastX = event.x
                lastY = event.y
                isDragging = false
                // 通知父View不要拦截事件
                parent.requestDisallowInterceptTouchEvent(true)
            }
            
            MotionEvent.ACTION_MOVE -> {
                val dx = event.x - lastX
                val dy = event.y - lastY
                
                // 移动View
                translationX += dx
                translationY += dy
                
                isDragging = true
            }
            
            MotionEvent.ACTION_UP -> {
                if (!isDragging) {
                    // 如果没有移动，视为点击
                    performClick()
                }
                parent.requestDisallowInterceptTouchEvent(false)
            }
            
            MotionEvent.ACTION_CANCEL -> {
                parent.requestDisallowInterceptTouchEvent(false)
            }
        }
        return true  // 消费所有事件
    }

    override fun performClick(): Boolean {
        super.performClick()
        // 处理点击逻辑
        return true
    }
}
```

---

## 选择建议

| 需求 | 推荐方案 | 原因 |
|------|---------|------|
| 给 Button 加按下效果 | `setOnTouchListener` | 简单，无需新类 |
| 监听 RecyclerView 滑动 | `setOnTouchListener` | 临时调试用 |
| 创建可拖拽悬浮窗 | 重写 `onTouchEvent` | 需要坐标计算和状态管理 |
| 自定义手势识别 | 重写 `onTouchEvent` | 需要完整的事件序列 |
| 拦截子 View 事件 | 重写 `onTouchEvent` + `onInterceptTouchEvent` | 只有父容器能做 |
| 游戏摇杆、画板 | 重写 `onTouchEvent` | 需要处理多点触控 |

---

## 常见陷阱

### 陷阱 1：onTouch 返回 true 后点击失效

```kotlin
button.setOnTouchListener { v, event ->
    // 处理触摸...
    true  // 糟糕！Button 的点击事件被拦截了！
}
```

**解决**：如果只想加效果不影响点击，返回 `false`

### 陷阱 2：忘记处理多点触控

```kotlin
// 错误：只处理了单指
override fun onTouchEvent(event: MotionEvent): Boolean {
    when (event.action) {  // 应该用 actionMasked
        ACTION_MOVE -> { ... }
    }
    return true
}

// 正确：处理多指（获取索引）
override fun onTouchEvent(event: MotionEvent): Boolean {
    when (event.actionMasked) {
        ACTION_POINTER_DOWN -> {
            val pointerIndex = event.actionIndex
            val pointerId = event.getPointerId(pointerIndex)
            // 处理第二根手指按下
        }
    }
    return true
}
```

### 陷阱 3：ACTION_UP 不触发

如果父 View 拦截了事件，子 View 可能收不到 `ACTION_UP`，需要：

```kotlin
// 在父 View 的 onInterceptTouchEvent 中
override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
    if (ev.action == MotionEvent.ACTION_UP) {
        // 通常不要在 UP 时拦截
        return false
    }
    // ...
}
```

---

## 总结

- **onTouch**：快餐式解决方案，适合简单场景
- **onTouchEvent**：正餐厅方案，适合需要精细控制的专业场景

**记住这个类比**：
- `setOnTouchListener` = 租房住，简单快捷但受限
- 重写 `onTouchEvent` = 买房装修，前期投入大但自由度极高

根据需求选择合适的方式，不要为了炫技而过度设计！
