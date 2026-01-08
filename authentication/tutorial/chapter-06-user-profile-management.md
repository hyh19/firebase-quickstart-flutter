# 第 6 章：用户资料管理

## Firebase 用户资料

Firebase User 对象包含以下可更新的属性：

- `displayName` - 显示名称
- `photoURL` - 头像 URL
- `email` - 邮箱地址（需要重新认证）

## 实现用户资料界面

### 1. 添加资料页面路由

```dart
routes: {
  // ... 其他路由
  '/profile': (context) {
    return ProfileScreen(
      appBar: AppBar(title: const Text('个人资料')),
      providers: const [],
      actions: [
        SignedOutAction(
          (context) {
            Navigator.of(context).pop();
          },
        ),
      ],
    );
  },
}
```

### 2. 更新首页添加资料按钮

```dart
TextButton(
  onPressed: () async {
    await Navigator.of(context).pushNamed('/profile');
    // 资料更新后刷新用户数据
    setState(() {});
  },
  child: const Text('个人资料'),
),
```

## ProfileScreen 详解

Firebase UI Auth 的 ProfileScreen 提供以下功能：

- 显示当前用户信息
- 允许编辑显示名称
- 提供密码重置选项
- 支持账户删除
- 集成登出功能

### 自定义 ProfileScreen

```dart
ProfileScreen(
  appBar: AppBar(
    title: const Text('个人资料'),
  ),
  providers: const [], // 如果有第三方登录提供商，在此添加
  actions: [
    SignedOutAction(
      (context) {
        // 处理登出后的导航
        Navigator.of(context).pop();
      },
    ),
  ],
  // 自定义子组件
  children: [
    const SizedBox(height: 20),
    // 添加自定义的用户信息显示
    Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            const Text(
              '账户信息',
              style: TextStyle(
                fontSize: 18,
                fontWeight: FontWeight.bold,
              ),
            ),
            const SizedBox(height: 16),
            Consumer<User?>(
              builder: (context, user, _) {
                if (user == null) return const SizedBox.shrink();
                return Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text('用户 ID：${user.uid}'),
                    Text('注册时间：${user.metadata.creationTime?.toString() ?? '未知'}'),
                    Text('最后登录：${user.metadata.lastSignInTime?.toString() ?? '未知'}'),
                  ],
                );
              },
            ),
          ],
        ),
      ),
    ),
  ],
)
```

## 手动实现资料管理

### 1. 创建自定义资料编辑组件

```dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter/material.dart';

class ProfileEditScreen extends StatefulWidget {
  const ProfileEditScreen({super.key});

  @override
  State<ProfileEditScreen> createState() => _ProfileEditScreenState();
}

class _ProfileEditScreenState extends State<ProfileEditScreen> {
  final _formKey = GlobalKey<FormState>();
  final _displayNameController = TextEditingController();
  final _photoUrlController = TextEditingController();
  bool _isLoading = false;

  @override
  void initState() {
    super.initState();
    final user = FirebaseAuth.instance.currentUser;
    if (user != null) {
      _displayNameController.text = user.displayName ?? '';
      _photoUrlController.text = user.photoURL ?? '';
    }
  }

  @override
  void dispose() {
    _displayNameController.dispose();
    _photoUrlController.dispose();
    super.dispose();
  }

  Future<void> _updateProfile() async {
    if (!_formKey.currentState!.validate()) return;

    setState(() => _isLoading = true);

    try {
      final user = FirebaseAuth.instance.currentUser;
      if (user == null) return;

      await user.updateDisplayName(_displayNameController.text.trim());
      await user.updatePhotoURL(_photoUrlController.text.trim());

      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('资料更新成功')),
        );
        Navigator.of(context).pop();
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('更新失败：$e')),
        );
      }
    } finally {
      if (mounted) {
        setState(() => _isLoading = false);
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('编辑资料'),
        actions: [
          TextButton(
            onPressed: _isLoading ? null : _updateProfile,
            child: _isLoading
                ? const SizedBox(
                    width: 20,
                    height: 20,
                    child: CircularProgressIndicator(strokeWidth: 2),
                  )
                : const Text('保存'),
          ),
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Form(
          key: _formKey,
          child: Column(
            children: [
              TextFormField(
                controller: _displayNameController,
                decoration: const InputDecoration(
                  labelText: '显示名称',
                  border: OutlineInputBorder(),
                ),
                validator: (value) {
                  if (value == null || value.trim().isEmpty) {
                    return '显示名称不能为空';
                  }
                  return null;
                },
              ),
              const SizedBox(height: 16),
              TextFormField(
                controller: _photoUrlController,
                decoration: const InputDecoration(
                  labelText: '头像 URL',
                  border: OutlineInputBorder(),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

## 密码管理

### 1. 密码重置

```dart
Future<void> _resetPassword() async {
  final user = FirebaseAuth.instance.currentUser;
  if (user?.email != null) {
    await FirebaseAuth.instance.sendPasswordResetEmail(
      email: user!.email!,
    );
    // 显示成功提示
  }
}
```

### 2. 修改密码

```dart
Future<void> _changePassword(String newPassword) async {
  final user = FirebaseAuth.instance.currentUser;
  await user?.updatePassword(newPassword);
}
```

## 练习

1. 使用 Firebase UI Auth 的 ProfileScreen
2. 创建自定义的用户资料编辑界面
3. 实现显示名称和头像 URL 的更新功能
4. 添加密码重置功能
5. 处理资料更新的错误情况

## 高级功能

### 邮箱地址更新

```dart
// 注意：更新邮箱需要重新认证
await user.verifyBeforeUpdateEmail(newEmail);
```

### 账户删除

```dart
Future<void> _deleteAccount() async {
  final user = FirebaseAuth.instance.currentUser;
  await user?.delete();
}
```

## 总结与检查清单

通过本章学习，你应该：

- [ ] 了解 Firebase 用户资料的结构
- [ ] 使用 ProfileScreen 组件
- [ ] 实现自定义资料编辑功能
- [ ] 处理显示名称和头像更新
- [ ] 了解密码管理的基本方法
- [ ] 处理资料更新的错误情况

下一章我们将学习认证状态监听的实现。
