# 第 2 章：项目设置和依赖

## 创建 Flutter 项目

首先，我们需要创建一个新的 Flutter 项目。在终端中运行以下命令：

```bash
flutter create storage_example
cd storage_example
```

## 配置 Firebase 项目

### 1. 创建 Firebase 项目

1. 访问 [Firebase 控制台](https://console.firebase.google.com/)
2. 点击"创建项目"或"Add project"
3. 输入项目名称，选择是否启用 Google Analytics
4. 等待项目创建完成

### 2. 启用 Storage

1. 在 Firebase 控制台左侧菜单中选择"Storage"
2. 点击"开始使用"
3. 选择存储桶位置（建议选择离用户最近的地区）
4. 配置安全规则（暂时保持默认测试规则）

### 3. 添加 Flutter 应用

1. 在项目概览页面点击 Flutter 图标添加应用
2. 输入应用包名（例如：`com.example.storage_example`）
3. 下载 `google-services.json` 文件（Android）或 `GoogleService-Info.plist` 文件（iOS）
4. 将配置文件放置到相应平台目录

## 配置依赖

在项目的 `pubspec.yaml` 文件中添加必要的依赖：

```yaml
name: storage_example
description: A Firebase Storage example app.
publish_to: 'none'
version: 1.0.0+1

environment:
  sdk: ">=2.17.0 <3.0.0"

dependencies:
  flutter:
    sdk: flutter

  # Firebase 核心库
  firebase_core: ^2.24.2

  # Firebase Storage
  firebase_storage: ^11.5.6

  # 图片选择器
  image_picker: ^1.0.5

  # 路径处理
  path_provider: ^2.0.10
  path: ^1.8.1

  # UI 组件
  cupertino_icons: ^1.0.2

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.1

flutter:
  uses-material-design: true
```

### 依赖说明

- **firebase_core**: Firebase 平台的核心依赖，所有 Firebase 服务都需要
- **firebase_storage**: Firebase Storage 的 Flutter SDK
- **image_picker**: 用于从设备相册选择图片
- **path_provider**: 获取设备存储路径
- **path**: 文件路径处理工具

## 安装依赖

添加依赖后，运行以下命令安装：

```bash
flutter pub get
```

## 配置平台特定设置

### Android 配置

1. 确保 `android/app/build.gradle` 中的 `minSdkVersion` 至少为 21：

```gradle
android {
    defaultConfig {
        minSdkVersion 21
        // ...
    }
}
```

1. 在 `android/app/src/main/AndroidManifest.xml` 中添加必要的权限：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- 添加存储权限 -->
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />

    <!-- 添加互联网权限 -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

    <application>
        <!-- ... 其他配置 ... -->
    </application>
</manifest>
```

### iOS 配置

1. 在 `ios/Runner/Info.plist` 中添加相册访问权限：

```xml
<key>NSPhotoLibraryUsageDescription</key>
<string>用于选择要上传的图片</string>

<key>NSCameraUsageDescription</key>
<string>用于拍摄照片进行上传</string>
```

## 验证配置

运行以下命令验证项目配置是否正确：

```bash
flutter doctor
flutter run
```

如果一切配置正确，应用应该能够正常启动。

## 项目结构

创建以下目录结构来组织代码：

```text
lib/
├── main.dart          # 应用入口
├── app.dart           # 应用主组件
├── firebase_options.dart  # Firebase 配置
└── src/
    └── page/
        ├── home.dart      # 主页面（上传功能）
        └── library.dart   # 图片库页面（下载展示）
```

## 小结

本章我们完成了项目的初始设置，包括：

- ✅ 创建 Flutter 项目
- ✅ 配置 Firebase 项目和 Storage
- ✅ 添加必要的依赖
- ✅ 配置平台特定权限
- ✅ 建立项目目录结构

在下一章中，我们将学习如何初始化 Firebase 并连接到 Storage 服务。
