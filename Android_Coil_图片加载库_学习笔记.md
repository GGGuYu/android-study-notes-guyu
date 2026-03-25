# Coil 图片加载库学习笔记

## 什么是 Coil

Coil 是一个 **Kotlin-first** 的图片加载库，专为 **Jetpack Compose** 设计，同时支持传统的 View 系统。

### 为什么选择 Coil？

- **Compose 原生支持**：提供 `AsyncImage` 等声明式 API，与 Compose 状态管理无缝集成
- **Kotlin 协程深度集成**：图片加载是挂起函数，天然支持取消和背压
- **轻量级**：约 200KB，比 Glide（600KB）更轻
- **高性能**：内存缓存、磁盘缓存、Bitmap 复用、智能解码

---

## Coil 核心优势

### 1. 与 Compose 的天然契合

```kotlin
// Coil 的 API 就像写 Compose 状态一样自然
AsyncImage(
    model = "https://example.com/image.jpg",
    contentDescription = "示例图片",
    modifier = Modifier.size(100.dp),
    placeholder = painterResource(R.drawable.placeholder),
    error = painterResource(R.drawable.error)
)
```

**优势**：
- 声明式 API，响应状态变化
- 自动处理重组，避免重复加载
- 项滑出屏幕时自动暂停加载

### 2. 双层缓存机制

#### 内存缓存（Memory Cache）

```kotlin
// 默认使用应用可用内存的 15%（不是 20%）
val imageLoader = ImageLoader.Builder(context)
    .memoryCache {
        MemoryCache.Builder(context)
            .maxSizePercent(0.15) // 默认 15%，可自定义
            .build()
    }
    .build()
```

**特点**：
- LruCache 实现，自动淘汰最少使用的图片
- 保存 Bitmap 对象，避免重复解码
- 支持配置最大内存占比

#### 磁盘缓存（Disk Cache）

```kotlin
// 基于 OkHttp 实现，支持 HTTP 缓存策略
val imageLoader = ImageLoader.Builder(context)
    .diskCache {
        DiskCache.Builder()
            .directory(context.cacheDir.resolve("image_cache"))
            .maxSizeBytes(512L * 1024 * 1024) // 512MB
            .build()
    }
    .build()
```

**特点**：
- 复用 OkHttp 的缓存机制
- 支持 HTTP 缓存头（Cache-Control）
- 持久化存储，应用重启后仍然有效

### 3. 智能解码与 Bitmap 复用

#### 自动尺寸匹配

```kotlin
// Coil 会自动判断 Bitmap 在当前配置下需要的格式和尺寸
AsyncImage(
    model = ImageRequest.Builder(context)
        .data("https://example.com/large_image.jpg")
        .size(200, 200) // 指定目标尺寸
        .scale(Scale.FIT) // 缩放模式
        .build(),
    contentDescription = null
)
```

**原理**：
- 根据目标尺寸采样（Subsampling），避免加载完整大图
- 自动选择合适的 Bitmap.Config（ARGB_8888、RGB_565 等）
- 减少内存占用，提升解码速度

#### Bitmap 复用池

```kotlin
// 自动复用 Bitmap，减少 GC 压力
val imageLoader = ImageLoader.Builder(context)
    .bitmapPoolEnabled(true) // 默认开启
    .build()
```

**原理**：
- 解码新图片时优先复用已存在的 Bitmap 内存
- 减少内存分配和垃圾回收

### 4. 防止布局抖动（Layout Shift）

```kotlin
// 使用占位图和错误图，避免内容跳动
AsyncImage(
    model = imageUrl,
    contentDescription = null,
    placeholder = painterResource(R.drawable.placeholder), // 加载中显示
    error = painterResource(R.drawable.error),             // 失败时显示
    fallback = painterResource(R.drawable.fallback)        // 数据为 null 时显示
)
```

**效果**：
- 图片加载前就有固定尺寸的占位
- 不会出现内容高度变化导致的界面跳动
- 提升用户体验

### 5. 预加载优化

```kotlin
// 配合 LazyColumn 预加载即将可见的图片
@Composable
fun ImageList(images: List<String>) {
    val context = LocalContext.current
    val imageLoader = context.imageLoader
    
    LazyColumn {
        itemsIndexed(images) { index, imageUrl ->
            AsyncImage(
                model = imageUrl,
                contentDescription = null,
                modifier = Modifier.fillMaxWidth()
            )
            
            // 预加载下一张图片
            if (index + 1 < images.size) {
                LaunchedEffect(Unit) {
                    imageLoader.enqueue(
                        ImageRequest.Builder(context)
                            .data(images[index + 1])
                            .size(Size.ORIGINAL)
                            .build()
                    )
                }
            }
        }
    }
}
```

---

## Coil vs 其他图片库

| 特性 | Coil | Glide | Picasso |
|------|------|-------|---------|
| **Compose 支持** | ⭐⭐⭐ 原生支持 | ⭐⭐ 需要包装 | ⭐ 不支持 |
| **协程集成** | ✅ 原生挂起函数 | ⚠️ 需要适配 | ❌ 回调式 |
| **包大小** | ~200KB | ~600KB | ~120KB |
| **内存缓存** | ✅ 15% 默认 | ✅ 可配置 | ⚠️ 较弱 |
| **磁盘缓存** | ✅ OkHttp 实现 | ✅ 自实现 | ⚠️ 无 |
| **GIF/WebP 支持** | ✅ 插件 | ✅ 内置 | ❌ 需扩展 |
| **Bitmap 复用** | ✅ 自动 | ✅ 自动 | ❌ 无 |

**选择建议**：
- **Compose 项目**：首选 Coil
- **传统 View + 复杂需求**：Glide 功能更全
- **简单需求 + 极简包**：Picasso

---

## 高级用法

### 自定义 ImageLoader

```kotlin
// 全局配置（推荐在 Application 中初始化）
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        val imageLoader = ImageLoader.Builder(this)
            .crossfade(true)                    // 淡入动画
            .crossfade(300)                     // 动画时长 300ms
            .placeholder(R.drawable.placeholder) // 全局占位图
            .error(R.drawable.error)            // 全局错误图
            .memoryCache {
                MemoryCache.Builder(this)
                    .maxSizePercent(0.2)        // 20% 内存
                    .build()
            }
            .diskCache {
                DiskCache.Builder()
                    .directory(cacheDir.resolve("coil_cache"))
                    .maxSizeBytes(256L * 1024 * 1024) // 256MB
                    .build()
            }
            .logger(DebugLogger())              // 调试日志
            .build()
        
        Coil.setImageLoader(imageLoader)
    }
}
```

### 图片变换（Transformations）

```kotlin
// 需要 coil-transformations 库
AsyncImage(
    model = ImageRequest.Builder(context)
        .data(url)
        .transformations(
            CircleCropTransformation(),      // 圆形裁剪
            RoundedCornersTransformation(16f), // 圆角
            GrayscaleTransformation()        // 灰度
        )
        .build(),
    contentDescription = null
)
```

### 监听加载状态

```kotlin
var imageState by remember { mutableStateOf<AsyncImagePainter.State>(AsyncImagePainter.State.Empty) }

AsyncImage(
    model = imageUrl,
    contentDescription = null,
    onState = { state ->
        imageState = state
        when (state) {
            is AsyncImagePainter.State.Loading -> {
                Log.d("Coil", "开始加载")
            }
            is AsyncImagePainter.State.Success -> {
                Log.d("Coil", "加载成功")
            }
            is AsyncImagePainter.State.Error -> {
                Log.e("Coil", "加载失败", state.result.throwable)
            }
            else -> {}
        }
    }
)
```

### 协程方式加载

```kotlin
// 在协程中加载图片
val scope = rememberCoroutineScope()

scope.launch {
    val request = ImageRequest.Builder(context)
        .data(url)
        .build()
    
    val result = context.imageLoader.execute(request)
    when (result) {
        is SuccessResult -> {
            val bitmap = (result.drawable as BitmapDrawable).bitmap
            // 处理 Bitmap
        }
        is ErrorResult -> {
            // 处理错误
        }
    }
}
```

---

## 性能优化最佳实践

### 1. LazyColumn 中的优化

```kotlin
@Composable
fun OptimizedImageList() {
    LazyColumn {
        items(
            items = images,
            key = { it.id } // 使用稳定的 key
        ) { image ->
            AsyncImage(
                model = ImageRequest.Builder(LocalContext.current)
                    .data(image.url)
                    .size(Size.ORIGINAL) // 让 Coil 自动计算最佳尺寸
                    .build(),
                contentDescription = null,
                contentScale = ContentScale.Crop,
                modifier = Modifier
                    .fillMaxWidth()
                    .aspectRatio(16f / 9f) // 固定宽高比，避免布局抖动
            )
        }
    }
}
```

### 2. 避免重复加载

```kotlin
// 使用 remember 缓存请求，避免重组时重复创建
@Composable
fun ImageItem(url: String) {
    val request = remember(url) {
        ImageRequest.Builder(context)
            .data(url)
            .crossfade(true)
            .build()
    }
    
    AsyncImage(
        model = request,
        contentDescription = null
    )
}
```

### 3. 内存优化

```kotlin
// 大图显示时降低质量
AsyncImage(
    model = ImageRequest.Builder(context)
        .data(largeImageUrl)
        .bitmapConfig(Bitmap.Config.RGB_565) // 减少内存占用（不支持透明）
        .size(1024, 1024) // 限制最大尺寸
        .build(),
    contentDescription = null
)
```

---

## 常见问题

### Q: 图片加载不出来？

```kotlin
// 检查网络权限和 HTTP 配置
ImageRequest.Builder(context)
    .data(url)
    .allowHardware(false) // 某些设备需要禁用硬件加速
    .build()
```

### Q: 如何加载本地资源？

```kotlin
// 本地文件
AsyncImage(model = File("/path/to/image.jpg"), ...)

// Drawable 资源
AsyncImage(model = R.drawable.image, ...)

// Uri
AsyncImage(model = "content://...".toUri(), ...)
```

### Q: 如何实现图片分享？

```kotlin
// 先下载到本地，再获取 File 分享
val request = ImageRequest.Builder(context)
    .data(url)
    .diskCachePolicy(CachePolicy.ENABLED)
    .build()

val result = imageLoader.execute(request)
if (result is SuccessResult) {
    val file = imageLoader.diskCache?.get(url)?.data?.toFile()
    // 分享 file
}
```

---

## 总结

| 使用场景 | 推荐做法 |
|---------|---------|
| Compose 列表图片 | 使用 `AsyncImage` + `LazyColumn` |
| 大图展示 | 指定 `size()` 限制解码尺寸 |
| 占位/错误处理 | 设置 `placeholder` 和 `error` |
| 内存优化 | 使用 `RGB_565` 或限制缓存大小 |
| 预加载 | 使用 `imageLoader.enqueue()` |
| 全局配置 | 在 Application 中初始化 `ImageLoader` |

---

## 参考资源

- [Coil 官方文档](https://coil-kt.github.io/coil/)
- [GitHub - coil-kt/coil](https://github.com/coil-kt/coil)
- [Jetpack Compose 图片加载指南](https://developer.android.com/jetpack/compose/graphics/images/loading)

---

*学习日期：2026年3月25日*
