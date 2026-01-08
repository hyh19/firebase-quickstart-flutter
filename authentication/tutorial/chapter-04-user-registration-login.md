# 第 4 章：用户注册和登录流程

## 用户注册流程

### 1. 邮箱验证处理

当新用户注册时，我们需要发送邮箱验证邮件：

```dart
AuthStateChangeAction(
  (context, state) {
    if (state is UserCreated) {
      final user = state.credential.user;
      if (user != null && !user.emailVerified) {
        user.sendEmailVerification();
      }
    }
    if (state is SignedIn || state is UserCreated) {
      Navigator.of(context).pushNamedAndRemoveUntil(
        '/home',
        (_) => false,
      );
    }
  },
)
```

### 2. 设置默认显示名称

为新用户设置默认显示名称：

```dart
if (state is UserCreated) {
  final user = state.credential.user;
  if (user != null && user.displayName == null && user.email != null) {
    final defaultDisplayName = user.email!.split('@')[0];
    user.updateDisplayName(defaultDisplayName);
  }
}
```

## 用户登录流程

### 1. 登录状态检测

Firebase UI Auth 自动处理登录流程，我们只需要监听状态变化：

```dart
if (state is SignedIn) {
  final user = state.user;
  // 用户已成功登录
  Navigator.of(context).pushNamedAndRemoveUntil(
    '/home',
    (_) => false,
  );
}
```

### 2. 处理登录错误

Firebase UI Auth 内置了错误处理，但我们也可以自定义错误显示：

```dart
SignInScreen(
  actions: [
    // ... 其他 actions
  ],
  // 自定义错误处理
  errorBuilder: (context, error, stackTrace, retry) {
    return Center(
      child: Column(
        children: [
          Text('登录失败：${error.toString()}'),
          ElevatedButton(
            onPressed: retry,
            child: const Text('重试'),
          ),
        ],
      ),
    );
  },
)
```

## 完整的认证流程

### 更新 app.dart

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
                  if (state is UserCreated) {
                    final user = state.credential.user;
                    if (user == null) return;

                    // 发送邮箱验证
                    if (!user.emailVerified) {
                      user.sendEmailVerification();
                    }

                    // 设置默认显示名称
                    if (user.displayName == null && user.email != null) {
                      final defaultDisplayName = user.email!.split('@')[0];
                      user.updateDisplayName(defaultDisplayName);
                    }
                  }

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

## 邮箱验证状态检查

### 在首页显示验证状态

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
              '欢迎，${user?.displayName ?? '用户'}！',
              style: Theme.of(context).textTheme.headlineMedium,
            ),
            const SizedBox(height: 20),
            if (user != null) ...[
              Text('邮箱：${user.email}'),
              Text('邮箱已验证：${user.emailVerified ? '是' : '否'}'),
              Text('用户 ID：${user.uid}'),
              const SizedBox(height: 20),
              if (!user.emailVerified) ...[
                ElevatedButton(
                  onPressed: () async {
                    await user.sendEmailVerification();
                    if (context.mounted) {
                      ScaffoldMessenger.of(context).showSnackBar(
                        const SnackBar(content: Text('验证邮件已发送')),
                      );
                    }
                  },
                  child: const Text('重新发送验证邮件'),
                ),
                const SizedBox(height: 10),
              ],
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

## 练习

1. 实现用户注册时的邮箱验证
2. 添加默认显示名称设置
3. 更新首页显示邮箱验证状态
4. 添加重新发送验证邮件功能
5. 测试完整的注册和登录流程

## 常见问题处理

### 邮箱验证邮件未收到

```dart
// 检查垃圾邮件文件夹
// 或者重新发送验证邮件
await user.sendEmailVerification();
```

### 显示名称更新失败

```dart
try {
  await user.updateDisplayName(displayName);
} catch (e) {
  print('更新显示名称失败：$e');
}
```

## 代码示例

完整的用户注册和登录流程已在上面的代码中展示。主要包括：

1. 监听 UserCreated 状态发送验证邮件
2. 设置默认显示名称
3. 在首页显示验证状态
4. 提供重新发送验证邮件的功能

## 总结与检查清单

通过本章学习，你应该：

- [ ] 理解用户注册和登录的完整流程
- [ ] 实现邮箱验证功能
- [ ] 设置用户的默认显示名称
- [ ] 在界面中显示邮箱验证状态
- [ ] 处理邮箱验证相关的用户交互

下一章我们将深入了解邮箱验证功能的实现细节。
