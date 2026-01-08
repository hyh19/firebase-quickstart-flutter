# 第 3 章：实现基础认证界面

## Firebase UI Auth 简介

Firebase UI Auth 是 Firebase 官方提供的预构建认证界面组件，可以快速实现常见的认证流程，包括登录、注册、密码重置等。

## 配置认证界面

### 1. 更新 app.dart

```dart
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
      initialRoute: '/',
      routes: {
        '/': (context) => SignInScreen(
          actions: [
            AuthStateChangeAction(
              (context, state) {
                if (state is SignedIn || state is UserCreated) {
                  Navigator.of(context).pushNamedAndRemoveUntil(
                    '/home',
                    (_) => false,
                  );
                }
              },
            ),
          ],
        ),
        '/home': (context) => const HomePage(),
      },
    );
  }
}
```

## SignInScreen 组件详解

### 基本用法

```dart
SignInScreen(
  actions: [
    // 处理认证状态变化
    AuthStateChangeAction((context, state) {
      if (state is SignedIn) {
        // 用户已登录，跳转到首页
        Navigator.of(context).pushNamed('/home');
      }
    }),
  ],
)
```

### 支持的操作类型

Firebase UI Auth 支持以下操作：

- `SignedIn` - 用户成功登录
- `UserCreated` - 新用户创建
- `SignedOut` - 用户登出
- `CredentialLinked` - 凭据链接
- `CredentialReceived` - 收到凭据

## 添加密码重置功能

### 更新路由配置

```dart
routes: {
  '/': (context) => SignInScreen(
    actions: [
      ForgotPasswordAction(
        (context, email) {
          Navigator.of(context).pushNamed(
            '/forgot-password',
            arguments: email,
          );
        },
      ),
      AuthStateChangeAction(
        (context, state) {
          if (state is SignedIn || state is UserCreated) {
            Navigator.of(context).pushNamedAndRemoveUntil(
              '/home',
              (_) => false,
            );
          }
        },
      ),
    ],
  ),
  '/forgot-password': (context) {
    final email = ModalRoute.of(context)?.settings.arguments as String;
    return ForgotPasswordScreen(email: email);
  },
  '/home': (context) => const HomePage(),
}
```

## 创建首页组件

### src/home_page.dart

```dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter/material.dart';

class HomePage extends StatelessWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    final user = FirebaseAuth.instance.currentUser;

    return Scaffold(
      appBar: AppBar(
        title: const Text('首页'),
        automaticallyImplyLeading: false,
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(
              '欢迎！',
              style: Theme.of(context).textTheme.headlineMedium,
            ),
            const SizedBox(height: 20),
            if (user != null) ...[
              Text('邮箱：${user.email}'),
              Text('用户 ID：${user.uid}'),
              const SizedBox(height: 20),
              ElevatedButton(
                onPressed: () async {
                  await FirebaseAuth.instance.signOut();
                },
                child: const Text('登出'),
              ),
            ],
          ],
        ),
      ),
    );
  }
}
```

## 运行应用

### 1. 启动 Firebase 模拟器

```bash
firebase emulators:start
```

### 2. 运行 Flutter 应用

```bash
flutter run
```

## 练习

1. 配置 SignInScreen 组件
2. 添加 AuthStateChangeAction 处理登录状态
3. 实现密码重置功能
4. 创建基本的 HomePage 组件
5. 测试应用运行

## 代码示例

完整的 app.dart 文件：

```dart
import 'package:firebase_ui_auth/firebase_ui_auth.dart';
import 'package:flutter/material.dart';

import 'home_page.dart';

class AuthApp extends StatelessWidget {
  const AuthApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      initialRoute: '/',
      routes: {
        '/': (context) {
          return SignInScreen(
            actions: [
              ForgotPasswordAction(
                (context, email) {
                  Navigator.of(context).pushNamed(
                    '/forgot-password',
                    arguments: email,
                  );
                },
              ),
              AuthStateChangeAction(
                (context, state) {
                  if (state is SignedIn || state is UserCreated) {
                    Navigator.of(context).pushNamedAndRemoveUntil(
                      '/home',
                      (_) => false,
                    );
                  }
                },
              ),
            ],
          );
        },
        '/forgot-password': (context) {
          final email = ModalRoute.of(context)?.settings.arguments as String;
          return ForgotPasswordScreen(email: email);
        },
        '/home': (context) => const HomePage(),
      },
    );
  }
}
```

## 总结与检查清单

通过本章学习，你应该：

- [ ] 了解 Firebase UI Auth 的使用方法
- [ ] 配置 SignInScreen 组件
- [ ] 实现认证状态变化处理
- [ ] 添加密码重置功能
- [ ] 创建基本的用户界面

下一章我们将深入了解用户注册和登录的具体流程。
