# Jetpack Compose - 状态提升详解

## 目录

- [1. 什么是状态提升](#1-什么是状态提升)
- [2. MVVM 架构下的状态管理](#2-mvvm-架构下的状态管理)
- [3. 状态持有层级](#3-状态持有层级)
- [4. 什么状态留在 Composable](#4-什么状态留在-composable)
- [5. 完整代码示例](#5-完整代码示例)

---

## 1. 什么是状态提升

状态提升（State Hoisting）是 Compose 中管理 UI 状态的核心模式。核心思想是：**子组件无状态，父组件持有状态**。

### 子组件：无状态，纯UI

```kotlin
// 子组件：无状态，纯展示
@Composable
fun UserCard(
    user: User,           // 数据通过参数传入
    onClick: () -> Unit   // 事件通过回调传出
) {
    // 没有 remember，没有状态，纯展示
    Card(onClick = onClick) {
        Text(user.name)
    }
}
```

子组件特点：
- 不持有状态（没有 `remember`）
- 只通过传参接收数据
- 只通过回调触发动作

---

## 2. MVVM 架构下的状态管理

在 MVVM 架构中，状态最终由 ViewModel 持有，Composables 只是**单向数据流**的传递者。

### 完整数据流

```kotlin
// ViewModel：持有业务状态
class UserViewModel : ViewModel() {
    private val _users = MutableStateFlow<List<User>>(emptyList())
    val users: StateFlow<List<User>> = _users.asStateFlow()
    
    fun loadUsers() { /* ... */ }
}

// 父组件（Screen）：连接 VM 和 UI
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val users by viewModel.users.collectAsStateWithLifecycle()
    
    // 状态在这里"解开"，然后传递给子组件
    UserList(
        users = users,
        onUserClick = { user -> /* 处理点击 */ }
    )
}

// 子组件列表：依然无状态
@Composable
fun UserList(
    users: List<User>,
    onUserClick: (User) -> Unit
) {
    LazyColumn {
        items(users) { user ->
            UserCard(
                user = user,
                onClick = { onUserClick(user) }
            )
        }
    }
}
```

### 关键原则

**子组件不应该有自己的 ViewModel**，它们应该是纯函数式的：

```kotlin
// ❌ 错误：子组件自己创建 ViewModel
@Composable
fun UserCard(userId: String) {
    val viewModel: UserDetailViewModel = hiltViewModel() // 不要这样做
    val user by viewModel.user.collectAsState()
    // ...
}

// ✅ 正确：通过参数接收数据
@Composable
fun UserCard(user: User, onClick: () -> Unit) {
    // 纯展示，逻辑交给父组件
}
```

---

## 3. 状态持有层级

| 层级 | 状态持有 | 职责 |
|------|---------|------|
| **子组件** | ❌ 无状态 | 纯展示，接收数据+回调 |
| **父组件 (Screen)** | ❌ 不直接持有 | 粘合剂，连接 VM ↔ UI |
| **ViewModel** | ✅ 持有业务状态 | 数据获取、业务逻辑、状态管理 |

状态提升的本质：**子组件越"蠢"，代码越可测试、可复用**。

---

## 4. 什么状态留在 Composable

只有**纯 UI 状态**（不涉及业务逻辑）可以保留在 Composable 中：

```kotlin
@Composable
fun ExpandableCard(
    title: String,
    content: @Composable () -> Unit
) {
    // 这个状态只在 UI 层使用，不需要保存到 ViewModel
    var isExpanded by remember { mutableStateOf(false) }
    
    Card {
        Text(title, modifier = Modifier.clickable { isExpanded = !isExpanded })
        if (isExpanded) content()
    }
}
```

### 区分标准

| 状态类型 | 存放位置 | 示例 |
|----------|----------|------|
| **业务状态** | ViewModel | 用户列表、购物车数据、登录状态 |
| **UI 状态** | Composable | 下拉框展开/收起、输入框焦点、动画状态 |

---

## 5. 完整代码示例

### ViewModel 层

```kotlin
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {

    private val _users = MutableStateFlow<List<User>>(emptyList())
    val users: StateFlow<List<User>> = _users.asStateFlow()
    
    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()

    fun loadUsers() {
        viewModelScope.launch {
            _isLoading.value = true
            _users.value = userRepository.getUsers()
            _isLoading.value = false
        }
    }
    
    fun onUserClick(user: User) {
        // 处理业务逻辑，如导航到详情页
    }
}
```

### Screen 层（父组件）

```kotlin
@Composable
fun UserListScreen(
    viewModel: UserListViewModel = hiltViewModel()
) {
    val users by viewModel.users.collectAsStateWithLifecycle()
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()

    UserListScreenContent(
        users = users,
        isLoading = isLoading,
        onUserClick = viewModel::onUserClick,
        onRefresh = viewModel::loadUsers
    )
}

@Composable
private fun UserListScreenContent(
    users: List<User>,
    isLoading: Boolean,
    onUserClick: (User) -> Unit,
    onRefresh: () -> Unit
) {
    Scaffold(
        topBar = { TopAppBar(title = { Text("用户列表") }) }
    ) { padding ->
        if (isLoading) {
            CircularProgressIndicator()
        } else {
            UserList(
                users = users,
                onUserClick = onUserClick,
                modifier = Modifier.padding(padding)
            )
        }
    }
}
```

### 子组件层

```kotlin
@Composable
fun UserList(
    users: List<User>,
    onUserClick: (User) -> Unit,
    modifier: Modifier = Modifier
) {
    LazyColumn(modifier = modifier) {
        items(users) { user ->
            UserCard(
                user = user,
                onClick = { onUserClick(user) }
            )
        }
    }
}

@Composable
fun UserCard(
    user: User,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Card(
        onClick = onClick,
        modifier = modifier.fillMaxWidth()
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Avatar(imageUrl = user.avatarUrl)
            Spacer(modifier = Modifier.width(12.dp))
            Column {
                Text(
                    text = user.name,
                    style = MaterialTheme.typography.titleMedium
                )
                Text(
                    text = user.email,
                    style = MaterialTheme.typography.bodyMedium
                )
            }
        }
    }
}

@Composable
fun Avatar(
    imageUrl: String,
    modifier: Modifier = Modifier
) {
    AsyncImage(
        model = imageUrl,
        contentDescription = "头像",
        modifier = modifier.size(48.dp)
    )
}
```

---

## 总结

状态提升配合 MVVM 架构的核心要点：

1. **子组件无状态**：只接收数据，触发回调
2. **父组件不解状态**：从 ViewModel 获取数据，传递给子组件
3. **ViewModel 统一持有业务状态**：数据来源、业务逻辑、状态管理
4. **UI 状态可留 Composable**：纯展示性的临时状态（展开/收起等）

**单向数据流**：ViewModel → Screen → 子组件 → 回调 → ViewModel

---

**创建时间**: 2025-03-19  
**标签**: Jetpack Compose, MVVM, 状态管理, 架构设计