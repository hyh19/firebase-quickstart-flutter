# 第 5 章：邮箱验证功能

## 邮箱验证的重要性

邮箱验证是现代应用安全性的重要组成部分，它可以：

1. **确认邮箱真实性**：确保用户提供的邮箱地址有效
2. **防止垃圾注册**：减少机器人和虚假账户
3. **提供安全保障**：为密码重置等功能提供安全基础
4. **改善用户体验**：提供更好的账户恢复选项

## Firebase 邮箱验证流程

### 1. 发送验证邮件

```dart
final user = FirebaseAuth.instance.currentUser;
if (user != null && !user.emailVerified) {
  await user.sendEmailVerification();
}
```

### 2. 检查验证状态

```dart
final user = FirebaseAuth.instance.currentUser;
final isVerified = user?.emailVerified ?? false;
```

### 3. 重新加载用户数据

```dart
await user?.reload();
final updatedUser = FirebaseAuth.instance.currentUser;
```

## 实现邮箱验证界面

### 1. 创建验证提示组件

```dart
class EmailVerificationBanner extends StatelessWidget {
  const EmailVerificationBanner({super.key});

  @override
  Widget build(BuildContext context) {
    final user = FirebaseAuth.instance.currentUser;

    if (user == null || user.emailVerified) {
      return const SizedBox.shrink();
    }

    return Container(
      color: Colors.orange.shade100,
      padding: const EdgeInsets.all(16),
      child: Row(
        children: [
          const Icon(Icons.warning, color: Colors.orange),
          const SizedBox(width: 16),
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                const Text(
                  '请验证您的邮箱',
                  style: TextStyle(fontWeight: FontWeight.bold),
                ),
                Text('我们已向 ${user.email} 发送验证邮件'),
              ],
            ),
          ),
          TextButton(
            onPressed: () async {
              try {
                await user.sendEmailVerification();
                if (context.mounted) {
                  ScaffoldMessenger.of(context).showSnackBar(
                    const SnackBar(content: Text('验证邮件已重新发送')),
                  );
                }
              } catch (e) {
                if (context.mounted) {
                  ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(content: Text('发送失败：$e')),
                  );
                }
              }
            },
            child: const Text('重新发送'),
          ),
        ],
      ),
    );
  }
}
```

### 2. 更新首页布局

```dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter/material.dart';

class HomePage extends StatefulWidget {
  const HomePage({super.key});

  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  User? _user;

  @override
  void initState() {
    super.initState();
    _user = FirebaseAuth.instance.currentUser;
  }

  Future<void> _refreshUser() async {
    await _user?.reload();
    setState(() {
      _user = FirebaseAuth.instance.currentUser;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('首页'),
        automaticallyImplyLeading: false,
        actions: [
          IconButton(
            onPressed: _refreshUser,
            icon: const Icon(Icons.refresh),
          ),
        ],
      ),
      body: Column(
        children: [
          const EmailVerificationBanner(),
          Expanded(
            child: Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Text(
                    '欢迎，${_user?.displayName ?? '用户'}！',
                    style: Theme.of(context).textTheme.headlineMedium,
                  ),
                  const SizedBox(height: 20),
                  if (_user != null) ...[
                    Text('邮箱：${_user!.email}'),
                    Text('邮箱已验证：${_user!.emailVerified ? '是' : '否'}'),
                    Text('用户 ID：${_user!.uid}'),
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
          ),
        ],
      ),
    );
  }
}
```

## 自动刷新验证状态

### 1. 使用 StreamBuilder 监听用户变化

```dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter/material.dart';

class HomePage extends StatelessWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    return StreamBuilder<User?>(
      stream: FirebaseAuth.instance.authStateChanges(),
      builder: (context, snapshot) {
        final user = snapshot.data;

        if (user == null) {
          return const Center(child: CircularProgressIndicator());
        }

        return Scaffold(
          appBar: AppBar(
            title: const Text('首页'),
            automaticallyImplyLeading: false,
          ),
          body: Column(
            children: [
              if (!user.emailVerified) const EmailVerificationBanner(),
              Expanded(
                child: Center(
                  child: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      Text(
                        '欢迎，${user.displayName ?? '用户'}！',
                        style: Theme.of(context).textTheme.headlineMedium,
                      ),
                      const SizedBox(height: 20),
                      Text('邮箱：${user.email}'),
                      Text('邮箱已验证：${user.emailVerified ? '是' : '否'}'),
                      Text('用户 ID：${user.uid}'),
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
              ),
            ],
          ),
        );
      },
    );
  }
}
```

## 处理验证邮件操作

### 1. 验证邮件中的链接

当用户点击验证邮件中的链接时，Firebase 会自动处理验证过程。应用需要监听用户状态变化来响应验证完成。

### 2. 自定义操作 URL

在 Firebase Console 中可以配置自定义的邮箱操作 URL：

1. 进入 Authentication > Templates
2. 选择 Email address verification
3. 配置自定义操作 URL

## 练习

1. 实现邮箱验证提示组件
2. 添加重新发送验证邮件功能
3. 使用 StreamBuilder 监听用户状态变化
4. 处理邮箱验证完成后的界面更新
5. 测试完整的邮箱验证流程

## 常见问题

### 验证邮件发送失败

```dart
try {
  await user.sendEmailVerification();
} catch (e) {
  // 处理发送失败的情况
  print('发送验证邮件失败：$e');
}
```

### 用户状态未及时更新

```dart
// 强制刷新用户数据
await user.reload();
final updatedUser = FirebaseAuth.instance.currentUser;
```

## 总结与检查清单

通过本章学习，你应该：

- [ ] 理解邮箱验证的重要性
- [ ] 实现发送验证邮件的功能
- [ ] 创建邮箱验证状态提示界面
- [ ] 使用 StreamBuilder 监听用户状态变化
- [ ] 处理验证邮件的重发逻辑
- [ ] 响应邮箱验证完成事件

下一章我们将学习用户资料管理的实现。
