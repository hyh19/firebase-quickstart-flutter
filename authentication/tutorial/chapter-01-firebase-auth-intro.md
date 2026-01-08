# 第 1 章：Firebase Authentication 简介与环境设置

## 简介

Firebase Authentication 是 Google 提供的用户认证服务，为移动和 Web 应用提供后端服务、易用的 SDK 和现成的 UI 库。本章将介绍 Firebase Authentication 的核心概念和开发环境设置。

## Firebase Authentication 核心概念

### 认证方式

Firebase Authentication 支持多种认证方式：

- **邮箱/密码认证**：最常用的认证方式
- **电话号码认证**：通过 SMS 验证码
- **第三方登录**：Google、Facebook、Twitter 等
- **匿名认证**：临时用户身份
- **自定义令牌**：服务器端生成的令牌

### 用户对象

每个认证用户都有以下属性：

```dart
class User {
  final String uid;           // 唯一用户 ID
  final String? email;        // 邮箱地址
  final String? displayName;  // 显示名称
  final String? photoURL;     // 头像 URL
  final bool emailVerified;   // 邮箱是否已验证
  final UserMetadata metadata; // 用户元数据
}
```

### 认证状态

Firebase Authentication 提供三种认证状态：

- **已认证**：用户已成功登录
- **未认证**：用户未登录或登录已过期
- **匿名**：用户以匿名方式登录

## 环境设置

### 1. 安装 Flutter

确保你已安装 Flutter SDK：

```bash
flutter --version
```

### 2. 安装 Firebase CLI

```bash
npm install -g firebase-tools
# 或使用其他包管理器
```

### 3. 登录 Firebase

```bash
firebase login
```

### 4. 安装 FlutterFire CLI

```bash
dart pub global activate flutterfire_cli
```

## Firebase 项目创建

### 1. 创建 Firebase 项目

访问 [Firebase Console](https://console.firebase.google.com/) 创建新项目。

### 2. 启用 Authentication

在 Firebase Console 中：

1. 选择你的项目
2. 点击左侧菜单的 "Authentication"
3. 进入 "Sign-in method" 标签页
4. 启用 "Email/Password" 提供商

## 核心依赖包

本教程将使用以下 Firebase 相关包：

```yaml
dependencies:
  firebase_core: ^2.24.1        # Firebase 核心功能
  firebase_auth: ^4.15.1        # Firebase 认证
  firebase_ui_auth: ^1.11.0     # Firebase UI 认证组件
```

## 练习

1. 创建一个新的 Firebase 项目
2. 启用 Email/Password 认证方式
3. 安装所需的开发工具（Flutter、Firebase CLI、FlutterFire CLI）

## 总结与检查清单

通过本章学习，你应该：

- [ ] 了解 Firebase Authentication 的基本概念
- [ ] 掌握不同认证方式的区别
- [ ] 完成开发环境设置
- [ ] 创建 Firebase 项目并启用认证服务
- [ ] 熟悉核心依赖包

下一章我们将开始创建 Flutter 项目并进行 Firebase 配置。
