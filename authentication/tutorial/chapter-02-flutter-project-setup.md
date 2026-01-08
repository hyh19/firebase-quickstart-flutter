# 第 2 章：创建 Flutter 项目并配置 Firebase

## 创建 Flutter 项目

### 1. 创建新项目

```bash
flutter create firebase_auth_tutorial
cd firebase_auth_tutorial
```

### 2. 配置 pubspec.yaml

添加必要的依赖包：

```yaml
name: firebase_auth_tutorial
description: Firebase Authentication Tutorial App

environment:
  sdk: ">=3.0.0 <4.0.0"

dependencies:
  flutter:
    sdk: flutter

  cupertino_icons: ^1.0.6
  firebase_core: ^2.24.1
  firebase_auth: ^4.15.1
  firebase_ui_auth: ^1.11.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.1

flutter:
  uses-material-design: true
```

### 3. 安装依赖

```bash
flutter pub get
```

## Firebase 项目配置

### 1. 初始化 FlutterFire

```bash
flutterfire configure
```

选择你的 Firebase 项目，并选择目标平台（iOS、Android）。

### 2. 生成的配置文件

FlutterFire 会自动生成以下文件：

- `lib/firebase_options.dart` - Firebase 配置选项
- `ios/Firebase/` - iOS Firebase 配置
- `android/app/google-services.json` - Android Firebase 配置

## 项目结构规划

创建以下目录结构：

```text
lib/
├── main.dart              # 应用入口
├── firebase_options.dart  # Firebase 配置
└── src/
    ├── app.dart           # 主应用组件
    └── home_page.dart     # 首页（登录后显示）
```

## 初始化 Firebase

### 修改 main.dart

```dart
import 'package:firebase_auth_tutorial/firebase_options.dart';
import 'package:firebase_auth/firebase_auth.dart' hide EmailAuthProvider;
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_ui_auth/firebase_ui_auth.dart';
import 'package:flutter/material.dart';

import 'src/app.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // 初始化 Firebase
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );

  // 配置认证提供商
  FirebaseUIAuth.configureProviders([
    EmailAuthProvider(),
  ]);

  runApp(const AuthApp());
}
```

## 创建应用组件

### src/app.dart

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
        '/': (context) => const SignInScreen(),
        '/home': (context) => const HomePage(),
      },
    );
  }
}
```

## 练习

1. 创建新的 Flutter 项目
2. 添加 Firebase 相关依赖
3. 使用 FlutterFire 配置 Firebase
4. 创建基本的项目结构
5. 初始化 Firebase 应用

## 代码示例

完整的 main.dart 文件：

```dart
// main.dart
import 'package:firebase_auth_tutorial/firebase_options.dart';
import 'package:firebase_auth/firebase_auth.dart' hide EmailAuthProvider;
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_ui_auth/firebase_ui_auth.dart';
import 'package:flutter/material.dart';

import 'src/app.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );

  FirebaseUIAuth.configureProviders([
    EmailAuthProvider(),
  ]);

  runApp(const AuthApp());
}
```

## 总结与检查清单

通过本章学习，你应该：

- [ ] 成功创建 Flutter 项目
- [ ] 配置正确的依赖包
- [ ] 使用 FlutterFire 配置 Firebase
- [ ] 初始化 Firebase 应用
- [ ] 创建基本的项目结构

下一章我们将实现认证用户界面。
