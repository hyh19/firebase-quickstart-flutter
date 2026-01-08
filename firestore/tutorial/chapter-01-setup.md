# 第 1 章：项目设置与 Firebase 配置

## 简介

本章将介绍如何设置 Flutter 项目并配置 Firebase，使其能够使用 Firestore 数据库。我们将从创建新的 Flutter 项目开始，然后逐步添加 Firebase 依赖和配置。

## Flutter 项目创建

首先，我们需要创建一个新的 Flutter 项目：

```bash
flutter create firestore_tutorial
cd firestore_tutorial
```

## 添加 Firebase 依赖

在 `pubspec.yaml` 文件中添加必要的 Firebase 依赖：

```yaml
dependencies:
  cloud_firestore: ^4.13.6
  firebase_auth: ^4.15.3
  firebase_core: ^2.24.2
  flutter:
    sdk: flutter
```

然后运行以下命令安装依赖：

```bash
flutter pub get
```

## Firebase 项目配置

### 1. 创建 Firebase 项目

1. 访问 [Firebase 控制台](https://console.firebase.google.com/)
2. 点击 "创建项目" 或 "Create a project"
3. 输入项目名称（如：friendly-eats-tutorial）
4. 启用 Google Analytics（可选）
5. 选择 Google Analytics 账户
6. 点击 "创建项目"

### 2. 启用 Firestore 数据库

1. 在 Firebase 控制台中，选择 "Firestore Database"
2. 点击 "创建数据库"
3. 选择 "以测试模式启动"（稍后我们会配置安全规则）
4. 选择数据库位置（建议选择离用户最近的位置）

### 3. 启用 Authentication

1. 在 Firebase 控制台中，选择 "Authentication"
2. 转到 "Sign-in method" 标签页
3. 找到 "Anonymous" 提供商
4. 点击启用
5. 保存更改

## 配置 Flutter 应用

### 1. 安装 Firebase CLI

```bash
npm install -g firebase-tools
# 或使用 Homebrew（macOS）
brew install firebase-cli
```

### 2. 登录 Firebase

```bash
firebase login
```

### 3. 初始化 FlutterFire

```bash
flutter pub add flutterfire_cli
flutterfire configure
```

在配置过程中：

- 选择你刚才创建的 Firebase 项目
- 选择目标平台（iOS、Android、Web 等）

这将在你的项目中生成 `firebase_options.dart` 文件。

## 应用初始化

在 `lib/main.dart` 中初始化 Firebase：

```dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter/material.dart';

import '../../firebase-firestore-tutorial/firebase_options.dart';

void main() async {
  // 确保 Flutter 绑定已初始化
  WidgetsFlutterBinding.ensureInitialized();

  // 初始化 Firebase
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );

  // 由于安全规则要求认证，我们使用匿名登录
  await FirebaseAuth.instance.signInAnonymously();

  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Firestore Tutorial',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: const HomePage(),
    );
  }
}
```

## Firestore 安全规则

在 Firestore 控制台的 "Rules" 标签页中，配置以下安全规则：

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // 允许所有已认证用户读写 restaurants 集合
    match /restaurants/{document=**} {
      allow read, write: if request.auth != null;
    }

    // 允许所有已认证用户读写 ratings 子集合
    match /restaurants/{restaurantid}/ratings/{document=**} {
      allow read, write: if request.auth != null;
    }
  }
}
```

## 验证配置

创建一个简单的测试来验证 Firebase 配置是否正确：

```dart
import 'package:cloud_firestore/cloud_firestore.dart';

class FirebaseTest {
  static Future<void> testConnection() async {
    try {
      // 测试写入
      await FirebaseFirestore.instance
          .collection('test')
          .doc('test_doc')
          .set({'message': 'Hello Firestore!', 'timestamp': Timestamp.now()});

      // 测试读取
      final doc = await FirebaseFirestore.instance
          .collection('test')
          .doc('test_doc')
          .get();

      print('Firebase 连接成功: ${doc.data()}');

      // 清理测试数据
      await FirebaseFirestore.instance
          .collection('test')
          .doc('test_doc')
          .delete();

    } catch (e) {
      print('Firebase 连接失败: $e');
    }
  }
}
```

在 `main.dart` 中调用测试：

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  await FirebaseAuth.instance.signInAnonymously();

  // 测试 Firebase 连接
  await FirebaseTest.testConnection();

  runApp(const MyApp());
}
```

## 运行应用

现在你可以运行应用来验证配置：

```bash
# Android
flutter run

# iOS
flutter run --device-id=<device_id>

# Web（需要 CORS 配置）
flutter run -d chrome --web-renderer html
```

## 常见问题

### 1. 平台配置错误

如果遇到平台特定的配置错误，请检查：

- iOS: `ios/Runner/GoogleService-Info.plist` 文件是否存在
- Android: `android/app/google-services.json` 文件是否存在
- Web: `web/index.html` 中的 Firebase 配置

### 2. 权限被拒绝

确保：

- 匿名认证已启用
- 安全规则允许已认证用户的访问
- 应用已正确初始化 Firebase

### 3. Web 平台 CORS 问题

在 `web/index.html` 中，确保使用 `--web-renderer html` 标志运行：

```bash
flutter run -d chrome --web-renderer html
```

## 总结与检查清单

通过本章的学习，你应该已经：

- [ ] 创建了 Flutter 项目并添加了 Firebase 依赖
- [ ] 在 Firebase 控制台创建了项目并启用了 Firestore 和 Authentication
- [ ] 使用 FlutterFire CLI 配置了应用
- [ ] 在代码中正确初始化了 Firebase 和匿名认证
- [ ] 配置了 Firestore 安全规则
- [ ] 验证了 Firebase 连接正常工作

## 下一步

在下一章中，我们将学习如何设计和实现数据模型，为餐厅和评论实体创建 Dart 类。
