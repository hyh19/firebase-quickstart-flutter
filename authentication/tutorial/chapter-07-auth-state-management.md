# 第 7 章：认证状态监听

## Firebase 认证状态

Firebase Authentication 提供以下状态：

- **已认证**：用户已成功登录，`FirebaseAuth.instance.currentUser != null`
- **未认证**：用户未登录或登录已过期
- **认证中**：正在处理登录/注册请求

## 使用 StreamBuilder 监听状态变化

### 1. 基本用法

```dart
StreamBuilder<User?>(
  stream: FirebaseAuth.instance.authStateChanges(),
  builder: (context, snapshot) {
    if (snapshot.connectionState == ConnectionState.waiting) {
      return const CircularProgressIndicator();
    }

    final user = snapshot.data;
    if (user == null) {
      return const SignInScreen();
    }

    return HomePage(user: user);
  },
)
```

### 2. 完整的应用结构

```dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:firebase_ui_auth/firebase_ui_auth.dart';
import 'package:flutter/material.dart';

import 'home_page.dart';

class AuthApp extends StatelessWidget {
  const AuthApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Firebase Auth Tutorial',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: StreamBuilder<User?>(
        stream: FirebaseAuth.instance.authStateChanges(),
        builder: (context, snapshot) {
          if (snapshot.connectionState == ConnectionState.waiting) {
            return const Scaffold(
              body: Center(
                child: CircularProgressIndicator(),
              ),
            );
          }

          final user = snapshot.data;
          if (user == null) {
            return SignInScreen(
              actions: [
                AuthStateChangeAction(
                  (context, state) {
                    if (state is SignedIn || state is UserCreated) {
                      // 状态变化会通过 StreamBuilder 自动处理
                    }
                  },
                ),
              ],
            );
          }

          return const HomePage();
        },
      ),
    );
  }
}
```

## 不同类型的状态监听

### 1. authStateChanges()

监听用户登录状态变化：

```dart
FirebaseAuth.instance.authStateChanges().listen((User? user) {
  if (user == null) {
    print('用户已登出');
  } else {
    print('用户已登录：${user.email}');
  }
});
```

### 2. userChanges()

监听用户数据变化（包括资料更新）：

```dart
FirebaseAuth.instance.userChanges().listen((User? user) {
  if (user != null) {
    print('用户数据已更新：${user.displayName}');
  }
});
```

### 3. idTokenChanges()

监听 ID Token 变化：

```dart
FirebaseAuth.instance.idTokenChanges().listen((User? user) {
  if (user != null) {
    // ID Token 已更新，可以刷新后端数据
  }
});
```

## 实现状态管理的 HomePage

### 1. 使用 StatefulWidget

```dart
import 'dart:async';

import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter/material.dart';

class HomePage extends StatefulWidget {
  const HomePage({super.key});

  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  late StreamSubscription<User?> _userSubscription;
  User? _user;

  @override
  void initState() {
    super.initState();
    _user = FirebaseAuth.instance.currentUser;

    // 监听用户状态变化
    _userSubscription = FirebaseAuth.instance.authStateChanges().listen((user) {
      if (mounted) {
        setState(() {
          _user = user;
        });
      }
    });
  }

  @override
  void dispose() {
    _userSubscription.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_user == null) {
      // 如果用户突然变为 null，返回登录界面
      WidgetsBinding.instance.addPostFrameCallback((_) {
        Navigator.of(context).pushNamedAndRemoveUntil(
          '/',
          (_) => false,
        );
      });
      return const Scaffold(
        body: Center(child: CircularProgressIndicator()),
      );
    }

    return Scaffold(
      appBar: AppBar(
        title: Text('欢迎，${_user!.displayName ?? '用户'}'),
        automaticallyImplyLeading: false,
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text('邮箱：${_user!.email}'),
            Text('用户 ID：${_user!.uid}'),
            const SizedBox(height: 20),
            ElevatedButton(
              onPressed: () async {
                await FirebaseAuth.instance.signOut();
              },
              child: const Text('登出'),
            ),
          ],
        ),
      ),
    );
  }
}
```

## 处理认证错误

### 1. 登录过期处理

```dart
FirebaseAuth.instance.authStateChanges().listen((user) async {
  if (user == null) {
    // 用户已登出，清除本地数据
    await _clearLocalData();
    // 导航到登录页面
    Navigator.of(context).pushNamedAndRemoveUntil('/', (_) => false);
  }
});
```

### 2. Token 刷新处理

```dart
FirebaseAuth.instance.idTokenChanges().listen((user) async {
  if (user != null) {
    // 获取新的 ID Token
    final idToken = await user.getIdToken();
    // 更新后端认证头
    await _updateBackendAuth(idToken);
  }
});
```

## 最佳实践

### 1. 避免内存泄漏

```dart
class _MyWidgetState extends State<MyWidget> {
  StreamSubscription? _subscription;

  @override
  void initState() {
    super.initState();
    _subscription = FirebaseAuth.instance.authStateChanges().listen(_onAuthStateChanged);
  }

  @override
  void dispose() {
    _subscription?.cancel();
    super.dispose();
  }

  void _onAuthStateChanged(User? user) {
    // 处理状态变化
  }
}
```

### 2. 处理连接状态

```dart
StreamBuilder<User?>(
  stream: FirebaseAuth.instance.authStateChanges(),
  builder: (context, snapshot) {
    if (snapshot.connectionState == ConnectionState.waiting) {
      return const LoadingScreen();
    }

    if (snapshot.hasError) {
      return ErrorScreen(error: snapshot.error);
    }

    final user = snapshot.data;
    return user == null ? const SignInScreen() : HomePage(user: user);
  },
)
```

## 练习

1. 使用 StreamBuilder 实现认证状态监听
2. 创建基于 StatefulWidget 的状态管理
3. 实现不同类型的状态监听（authStateChanges、userChanges、idTokenChanges）
4. 处理登录过期和 Token 刷新的情况
5. 优化内存管理和错误处理

## 调试技巧

### 1. 打印认证状态

```dart
FirebaseAuth.instance.authStateChanges().listen((user) {
  print('Auth state changed: ${user?.email ?? 'null'}');
});
```

### 2. 检查当前用户

```dart
final user = FirebaseAuth.instance.currentUser;
print('Current user: ${user?.toString()}');
```

## 总结与检查清单

通过本章学习，你应该：

- [ ] 了解 Firebase 认证状态的不同类型
- [ ] 使用 StreamBuilder 监听状态变化
- [ ] 实现基于 StatefulWidget 的状态管理
- [ ] 处理认证状态变化事件
- [ ] 管理 Stream 订阅的生命周期
- [ ] 处理认证错误和异常情况

下一章我们将学习如何使用 Firebase 模拟器进行本地测试。
