# 第 3 章：Firebase 初始化

## Firebase 配置选项

在开始使用 Firebase Storage 之前，我们需要生成 Firebase 配置选项文件。这个文件包含了连接到 Firebase 项目的必要信息。

### 生成配置文件

当我们在 Firebase 控制台添加 Flutter 应用时，控制台会为我们生成配置文件：

- **Android**: `google-services.json`
- **iOS**: `GoogleService-Info.plist`

这些文件包含了项目的 API 密钥、项目 ID 等敏感信息，切勿将其提交到版本控制系统。

### Firebase Options 类

FlutterFire 提供了 `firebase_options.dart` 文件来安全地处理这些配置。我们可以使用以下命令生成：

```bash
flutterfire configure
```

或者手动创建 `firebase_options.dart` 文件：

```dart
import 'package:firebase_core/firebase_core.dart' show FirebaseOptions;
import 'package:flutter/foundation.dart'
    show defaultTargetPlatform, kIsWeb, TargetPlatform;

class DefaultFirebaseOptions {
  static FirebaseOptions get currentPlatform {
    if (kIsWeb) {
      return web;
    }
    switch (defaultTargetPlatform) {
      case TargetPlatform.android:
        return android;
      case TargetPlatform.iOS:
        return ios;
      case TargetPlatform.macOS:
        return macos;
      default:
        throw UnsupportedError(
          'DefaultFirebaseOptions are not supported for this platform.',
        );
    }
  }

  static const FirebaseOptions web = FirebaseOptions(
    apiKey: 'your-web-api-key',
    appId: 'your-web-app-id',
    messagingSenderId: 'your-sender-id',
    projectId: 'your-project-id',
    authDomain: 'your-project.firebaseapp.com',
    storageBucket: 'your-project.appspot.com',
  );

  static const FirebaseOptions android = FirebaseOptions(
    apiKey: 'your-android-api-key',
    appId: 'your-android-app-id',
    messagingSenderId: 'your-sender-id',
    projectId: 'your-project-id',
    storageBucket: 'your-project.appspot.com',
  );

  static const FirebaseOptions ios = FirebaseOptions(
    apiKey: 'your-ios-api-key',
    appId: 'your-ios-app-id',
    messagingSenderId: 'your-sender-id',
    projectId: 'your-project-id',
    storageBucket: 'your-project.appspot.com',
  );

  static const FirebaseOptions macos = FirebaseOptions(
    apiKey: 'your-macos-api-key',
    appId: 'your-macos-app-id',
    messagingSenderId: 'your-sender-id',
    projectId: 'your-project-id',
    storageBucket: 'your-project.appspot.com',
  );
}
```

## 初始化 Firebase

在 Flutter 应用中，我们需要在 `main()` 函数中初始化 Firebase。这是一个异步操作，必须在 `runApp()` 之前完成。

### 基本初始化

创建 `main.dart` 文件：

```dart
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter/material.dart';

import '../../firebase-storage-example/app.dart';
import '../../firebase-storage-example/firebase_options.dart';

void main() async {
  // 确保 Flutter 绑定已初始化
  WidgetsFlutterBinding.ensureInitialized();

  // 初始化 Firebase
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );

  // 启动应用
  runApp(const App());
}
```

### 初始化步骤详解

1. **WidgetsFlutterBinding.ensureInitialized()**
   - 初始化 Flutter 引擎与平台通道的连接
   - 必须在调用任何平台相关代码之前执行

2. **Firebase.initializeApp()**
   - 初始化 Firebase 应用实例
   - 使用平台特定的配置选项
   - 返回一个 `FirebaseApp` 实例

3. **错误处理**
   - 如果初始化失败，应用将无法使用 Firebase 服务
   - 建议添加适当的错误处理逻辑

### 高级初始化选项

#### 多 Firebase 项目

如果你的应用需要连接多个 Firebase 项目：

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // 初始化默认项目
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );

  // 初始化第二个项目
  await Firebase.initializeApp(
    name: 'secondary',
    options: SecondaryFirebaseOptions.currentPlatform,
  );

  runApp(const App());
}
```

#### 自定义 FirebaseApp 实例

```dart
final secondaryApp = Firebase.app('secondary');
final secondaryStorage = FirebaseStorage.instanceFor(app: secondaryApp);
```

## 验证初始化

我们可以通过以下方式验证 Firebase 是否正确初始化：

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  try {
    await Firebase.initializeApp(
      options: DefaultFirebaseOptions.currentPlatform,
    );
    print('Firebase 初始化成功');
  } catch (e) {
    print('Firebase 初始化失败: $e');
  }

  runApp(const App());
}
```

## 应用架构

现在我们需要创建应用的主架构。在 `app.dart` 中定义路由和主题：

```dart
import 'package:flutter/material.dart';
import 'package:storage_example/src/page/home.dart';
import 'package:storage_example/src/page/library.dart';

class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Firebase Storage 示例',
      theme: ThemeData(
        primarySwatch: Colors.blue,
        useMaterial3: true,
      ),
      initialRoute: '/',
      routes: {
        '/': (context) => const HomeScreen(),
        '/library': (context) => const LibraryPage(),
      },
    );
  }
}
```

## 核心组件介绍

我们的应用包含两个主要页面：

1. **HomeScreen** (`home.dart`): 图片选择和上传界面
2. **LibraryPage** (`library.dart`): 已上传图片的浏览界面

## 小结

本章我们学习了如何在 Flutter 应用中初始化 Firebase：

- ✅ 配置 Firebase 选项
- ✅ 在 main() 函数中初始化 Firebase
- ✅ 处理多项目场景
- ✅ 验证初始化状态
- ✅ 建立应用的基本架构

现在 Firebase 已经正确初始化，我们可以在下一章中实现图片上传功能了。
