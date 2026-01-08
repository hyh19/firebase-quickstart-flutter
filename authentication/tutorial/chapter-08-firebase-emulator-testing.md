# 第 8 章：使用 Firebase 模拟器测试

## Firebase 模拟器简介

Firebase 模拟器套件允许你在本地开发环境中测试 Firebase 应用，无需连接到生产 Firebase 服务。

### 主要优势

- **离线开发**：无需网络连接
- **快速测试**：比生产环境更快
- **安全测试**：不会影响生产数据
- **成本控制**：避免产生费用
- **CI/CD 集成**：便于自动化测试

## 配置 Firebase 模拟器

### 1. 安装 Firebase CLI

```bash
npm install -g firebase-tools
```

### 2. 登录 Firebase

```bash
firebase login
```

### 3. 初始化项目

```bash
firebase init emulators
```

选择 Authentication 和 Firestore 模拟器。

### 4. firebase.json 配置

```json
{
  "emulators": {
    "auth": {
      "port": 9099
    },
    "firestore": {
      "port": 8080
    },
    "ui": {
      "enabled": true,
      "port": 4000
    }
  }
}
```

## 在 Flutter 应用中使用模拟器

### 1. 修改 main.dart

```dart
import 'package:firebase_auth/firebase_auth.dart' hide EmailAuthProvider;
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_ui_auth/firebase_ui_auth.dart';
import 'package:flutter/material.dart';

import 'firebase_options.dart';
import 'src/app.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );

  // 配置认证提供商
  FirebaseUIAuth.configureProviders([
    EmailAuthProvider(),
  ]);

  // 连接到本地模拟器
  await FirebaseAuth.instance.useAuthEmulator('localhost', 9099);

  // 为了测试方便，自动登出当前用户
  await FirebaseAuth.instance.signOut();

  runApp(const AuthApp());
}
```

### 2. 启动模拟器

```bash
firebase emulators:start
```

### 3. 运行应用

```bash
flutter run
```

## 模拟器 UI 界面

### 访问模拟器 UI

打开浏览器访问 `http://localhost:4000` 可以查看：

- **Authentication**：查看和管理模拟用户
- **Firestore**：查看和管理模拟数据
- **Logs**：查看模拟器日志

### 认证模拟器功能

在 Authentication 标签页中可以：

- 查看所有模拟用户
- 手动添加测试用户
- 删除用户
- 查看用户详细信息
- 模拟邮箱验证

## 编写测试代码

### 1. 单元测试示例

```dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  setUpAll(() async {
    // 设置测试环境
    await FirebaseAuth.instance.useAuthEmulator('localhost', 9099);
  });

  tearDownAll(() async {
    // 清理测试数据
    await FirebaseAuth.instance.signOut();
  });

  test('用户注册测试', () async {
    final email = 'test@example.com';
    final password = 'password123';

    // 创建测试用户
    final credential = await FirebaseAuth.instance.createUserWithEmailAndPassword(
      email: email,
      password: password,
    );

    expect(credential.user, isNotNull);
    expect(credential.user!.email, email);
  });

  test('用户登录测试', () async {
    final email = 'test@example.com';
    final password = 'password123';

    // 登录用户
    final credential = await FirebaseAuth.instance.signInWithEmailAndPassword(
      email: email,
      password: password,
    );

    expect(credential.user, isNotNull);
    expect(credential.user!.email, email);
  });
}
```

### 2. 集成测试示例

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('完整的认证流程测试', (WidgetTester tester) async {
    // 启动应用
    await tester.pumpWidget(const MyApp());

    // 等待 Firebase 初始化
    await tester.pumpAndSettle();

    // 找到邮箱输入框
    final emailField = find.byType(TextFormField).first;
    await tester.enterText(emailField, 'test@example.com');

    // 找到密码输入框
    final passwordField = find.byType(TextFormField).last;
    await tester.enterText(passwordField, 'password123');

    // 点击注册按钮
    final signUpButton = find.text('注册');
    await tester.tap(signUpButton);

    // 等待导航到首页
    await tester.pumpAndSettle();

    // 验证用户已登录
    expect(find.text('欢迎'), findsOneWidget);
  });
}
```

## 模拟器数据管理

### 1. 导出/导入数据

```bash
# 导出模拟器数据
firebase emulators:export ./emulator-data

# 导入模拟器数据
firebase emulators:start --import=./emulator-data
```

### 2. 清除数据

```bash
# 停止模拟器
firebase emulators:stop

# 删除数据目录或重新启动
rm -rf firebase-emulator-data
```

## 常见测试场景

### 1. 邮箱验证测试

```dart
test('邮箱验证测试', () async {
  final user = FirebaseAuth.instance.currentUser;
  expect(user!.emailVerified, false);

  // 在模拟器 UI 中手动验证邮箱
  // 或使用模拟器 API
  await user.sendEmailVerification();

  // 刷新用户数据
  await user.reload();
  expect(user.emailVerified, true);
});
```

### 2. 密码重置测试

```dart
test('密码重置测试', () async {
  const email = 'test@example.com';

  // 发送密码重置邮件
  await FirebaseAuth.instance.sendPasswordResetEmail(email: email);

  // 验证邮件已发送（在模拟器中检查）
});
```

### 3. 用户资料更新测试

```dart
test('用户资料更新测试', () async {
  final user = FirebaseAuth.instance.currentUser!;

  const newDisplayName = '新显示名称';
  await user.updateDisplayName(newDisplayName);

  await user.reload();
  expect(user.displayName, newDisplayName);
});
```

## CI/CD 集成

### 1. GitHub Actions 示例

```yaml
name: Test with Firebase Emulators

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2
    - uses: actions/setup-node@v2
      with:
        node-version: '16'

    - name: Setup Flutter
      uses: subosito/flutter-action@v2
      with:
        flutter-version: '3.0.0'

    - name: Install Firebase CLI
      run: npm install -g firebase-tools

    - name: Start Firebase Emulators
      run: firebase emulators:start --project=test-project &
      env:
        FIREBASE_AUTH_EMULATOR_HOST: localhost:9099

    - name: Run Flutter tests
      run: flutter test
```

## 练习

1. 配置 Firebase 模拟器环境
2. 修改应用代码使用模拟器
3. 在模拟器 UI 中查看和管理用户
4. 编写单元测试验证认证功能
5. 创建集成测试覆盖完整流程
6. 配置 CI/CD 流水线使用模拟器

## 故障排除

### 模拟器连接失败

```bash
# 检查模拟器是否正在运行
firebase emulators:list

# 检查端口是否被占用
lsof -i :9099
```

### 测试数据清理

```dart
// 在测试开始前清理
await FirebaseAuth.instance.signOut();

// 删除测试用户
final users = await FirebaseAuth.instance.fetchSignInMethodsForEmail(email);
if (users.isNotEmpty) {
  // 需要管理员权限删除用户
}
```

## 总结与检查清单

通过本章学习，你应该：

- [ ] 了解 Firebase 模拟器的优势和用途
- [ ] 配置本地 Firebase 模拟器环境
- [ ] 修改应用代码连接到模拟器
- [ ] 使用模拟器 UI 界面管理测试数据
- [ ] 编写单元测试和集成测试
- [ ] 配置 CI/CD 流水线使用模拟器
- [ ] 处理常见的模拟器使用问题

恭喜！你已经完成了 Firebase Authentication Flutter 教程系列的学习。通过这个教程，你学会了如何在 Flutter 应用中集成完整的用户认证系统，包括用户注册、登录、邮箱验证、资料管理以及本地测试等核心功能。

你可以基于这些知识继续扩展应用，添加更多 Firebase 服务，如 Cloud Firestore、Cloud Storage 等。
