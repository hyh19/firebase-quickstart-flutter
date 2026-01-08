# 第 4 章：认证集成与安全规则

## 简介

本章将介绍如何在 Flutter 应用中集成 Firebase Authentication，并配置相应的 Firestore 安全规则。我们将学习匿名认证、用户状态管理，以及如何根据用户权限控制数据访问。

## Firebase Authentication 基础

### 认证方式

Firebase 提供了多种认证方式：

- **匿名认证**：无需用户提供个人信息
- **邮箱/密码认证**：传统的用户名密码方式
- **Google 登录**：使用 Google 账户
- **Apple 登录**：使用 Apple ID
- **电话号码认证**：使用短信验证码

### 为什么需要认证

1. **安全规则**：Firestore 安全规则通常要求用户已认证
2. **用户数据隔离**：区分不同用户的数据
3. **审核追踪**：记录操作的用户信息

## 实现认证服务

创建 `lib/services/auth_service.dart`：

```dart
import 'package:firebase_auth/firebase_auth.dart';

/// 认证服务类
class AuthService {
  final FirebaseAuth _auth = FirebaseAuth.instance;

  /// 获取当前用户流
  Stream<User?> get user => _auth.authStateChanges();

  /// 获取当前用户
  User? get currentUser => _auth.currentUser;

  /// 判断用户是否已登录
  bool get isAuthenticated => currentUser != null;

  /// 匿名登录
  Future<UserCredential> signInAnonymously() async {
    try {
      final userCredential = await _auth.signInAnonymously();
      print('匿名登录成功: ${userCredential.user?.uid}');
      return userCredential;
    } catch (e) {
      print('匿名登录失败: $e');
      rethrow;
    }
  }

  /// 邮箱密码注册
  Future<UserCredential> signUpWithEmailAndPassword(
    String email,
    String password,
  ) async {
    try {
      final userCredential = await _auth.createUserWithEmailAndPassword(
        email: email,
        password: password,
      );
      print('注册成功: ${userCredential.user?.email}');
      return userCredential;
    } catch (e) {
      print('注册失败: $e');
      rethrow;
    }
  }

  /// 邮箱密码登录
  Future<UserCredential> signInWithEmailAndPassword(
    String email,
    String password,
  ) async {
    try {
      final userCredential = await _auth.signInWithEmailAndPassword(
        email: email,
        password: password,
      );
      print('登录成功: ${userCredential.user?.email}');
      return userCredential;
    } catch (e) {
      print('登录失败: $e');
      rethrow;
    }
  }

  /// 登出
  Future<void> signOut() async {
    try {
      await _auth.signOut();
      print('登出成功');
    } catch (e) {
      print('登出失败: $e');
      rethrow;
    }
  }

  /// 更新用户资料
  Future<void> updateProfile({
    String? displayName,
    String? photoURL,
  }) async {
    try {
      await currentUser?.updateDisplayName(displayName);
      await currentUser?.updatePhotoURL(photoURL);
      print('用户资料更新成功');
    } catch (e) {
      print('更新用户资料失败: $e');
      rethrow;
    }
  }

  /// 发送密码重置邮件
  Future<void> sendPasswordResetEmail(String email) async {
    try {
      await _auth.sendPasswordResetEmail(email: email);
      print('密码重置邮件已发送');
    } catch (e) {
      print('发送密码重置邮件失败: $e');
      rethrow;
    }
  }

  /// 删除当前用户账户
  Future<void> deleteAccount() async {
    try {
      await currentUser?.delete();
      print('账户删除成功');
    } catch (e) {
      print('删除账户失败: $e');
      rethrow;
    }
  }
}
```

## 创建认证状态管理

创建 `lib/providers/auth_provider.dart` 来管理认证状态：

```dart
import 'package:flutter/foundation.dart';

import '../../models/user_model.dart';
import '../../services/auth_service.dart';

/// 认证状态枚举
enum AuthState {
  initial,
  authenticated,
  unauthenticated,
  loading,
  error,
}

/// 认证提供者
class AuthProvider with ChangeNotifier {
  final AuthService _authService;

  AuthProvider(this._authService) {
    _init();
  }

  AuthState _state = AuthState.initial;
  UserModel? _user;
  String? _errorMessage;

  AuthState get state => _state;
  UserModel? get user => _user;
  String? get errorMessage => _errorMessage;
  bool get isAuthenticated => _state == AuthState.authenticated;

  void _init() {
    // 监听认证状态变化
    _authService.user.listen(_onAuthStateChanged);
  }

  void _onAuthStateChanged(User? firebaseUser) {
    if (firebaseUser != null) {
      _user = UserModel.fromFirebaseUser(firebaseUser);
      _state = AuthState.authenticated;
      _errorMessage = null;
    } else {
      _user = null;
      _state = AuthState.unauthenticated;
    }
    notifyListeners();
  }

  /// 匿名登录
  Future<void> signInAnonymously() async {
    try {
      _state = AuthState.loading;
      notifyListeners();

      await _authService.signInAnonymously();
      // 状态变化会通过监听器自动更新
    } catch (e) {
      _state = AuthState.error;
      _errorMessage = _getErrorMessage(e);
      notifyListeners();
    }
  }

  /// 邮箱密码登录
  Future<void> signInWithEmailAndPassword(String email, String password) async {
    try {
      _state = AuthState.loading;
      notifyListeners();

      await _authService.signInWithEmailAndPassword(email, password);
    } catch (e) {
      _state = AuthState.error;
      _errorMessage = _getErrorMessage(e);
      notifyListeners();
    }
  }

  /// 邮箱密码注册
  Future<void> signUpWithEmailAndPassword(String email, String password) async {
    try {
      _state = AuthState.loading;
      notifyListeners();

      await _authService.signUpWithEmailAndPassword(email, password);
    } catch (e) {
      _state = AuthState.error;
      _errorMessage = _getErrorMessage(e);
      notifyListeners();
    }
  }

  /// 登出
  Future<void> signOut() async {
    try {
      await _authService.signOut();
    } catch (e) {
      _errorMessage = _getErrorMessage(e);
      notifyListeners();
    }
  }

  /// 更新用户资料
  Future<void> updateProfile({
    String? displayName,
    String? photoURL,
  }) async {
    try {
      await _authService.updateProfile(
        displayName: displayName,
        photoURL: photoURL,
      );

      // 重新获取用户信息
      if (_authService.currentUser != null) {
        _user = UserModel.fromFirebaseUser(_authService.currentUser!);
        notifyListeners();
      }
    } catch (e) {
      _errorMessage = _getErrorMessage(e);
      notifyListeners();
    }
  }

  String _getErrorMessage(Object error) {
    if (error is FirebaseAuthException) {
      switch (error.code) {
        case 'user-not-found':
          return '用户不存在';
        case 'wrong-password':
          return '密码错误';
        case 'email-already-in-use':
          return '邮箱已被注册';
        case 'weak-password':
          return '密码强度太弱';
        case 'invalid-email':
          return '邮箱格式无效';
        case 'operation-not-allowed':
          return '该登录方式未启用';
        case 'user-disabled':
          return '用户已被禁用';
        default:
          return '认证失败: ${error.message}';
      }
    }
    return '未知错误: $error';
  }
}
```

## 创建用户模型

创建 `lib/models/user_model.dart`：

```dart
import 'package:firebase_auth/firebase_auth.dart';

/// 用户模型
class UserModel {
  final String uid;
  final String? email;
  final String? displayName;
  final String? photoURL;
  final bool isAnonymous;
  final DateTime? creationTime;
  final DateTime? lastSignInTime;

  UserModel({
    required this.uid,
    this.email,
    this.displayName,
    this.photoURL,
    required this.isAnonymous,
    this.creationTime,
    this.lastSignInTime,
  });

  /// 从 Firebase User 创建 UserModel
  factory UserModel.fromFirebaseUser(User user) {
    return UserModel(
      uid: user.uid,
      email: user.email,
      displayName: user.displayName,
      photoURL: user.photoURL,
      isAnonymous: user.isAnonymous,
      creationTime: user.metadata.creationTime,
      lastSignInTime: user.metadata.lastSignInTime,
    );
  }

  /// 获取用户显示名称
  String get displayNameOrDefault {
    if (displayName != null && displayName!.isNotEmpty) {
      return displayName!;
    }
    if (email != null) {
      return email!.split('@').first;
    }
    return '匿名用户';
  }

  /// 获取用户头像URL
  String? get avatarUrl {
    if (photoURL != null && photoURL!.isNotEmpty) {
      return photoURL;
    }
    // 返回默认头像URL或null
    return null;
  }

  /// 判断是否为新用户（注册时间在24小时内）
  bool get isNewUser {
    if (creationTime == null) return false;
    final now = DateTime.now();
    final difference = now.difference(creationTime!);
    return difference.inHours < 24;
  }

  @override
  String toString() {
    return 'UserModel(uid: $uid, displayName: $displayName, email: $email, isAnonymous: $isAnonymous)';
  }

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is UserModel && other.uid == uid;
  }

  @override
  int get hashCode => uid.hashCode;
}
```

## 配置 Firestore 安全规则

更新 Firestore 安全规则以支持不同的认证状态：

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // 辅助函数：检查用户是否已认证
    function isAuthenticated() {
      return request.auth != null;
    }

    // 辅助函数：检查用户是否为文档所有者
    function isOwner(userId) {
      return isAuthenticated() && request.auth.uid == userId;
    }

    // 辅助函数：检查是否为数据创建操作
    function isCreate() {
      return request.method == 'create';
    }

    // 辅助函数：检查是否为数据更新操作
    function isUpdate() {
      return request.method == 'update';
    }

    // 餐厅集合规则
    match /restaurants/{restaurantId} {
      // 允许所有已认证用户读取餐厅
      allow read: if isAuthenticated();

      // 只允许已认证用户创建餐厅
      allow create: if isAuthenticated() &&
                       request.resource.data.keys().hasAll(['name', 'category']) &&
                       request.resource.data.name is string &&
                       request.resource.data.name.size() > 0;

      // 只允许餐厅创建者或管理员更新餐厅
      allow update: if isAuthenticated() &&
                       // 只能更新特定字段
                       request.resource.data.diff(resource.data).affectedKeys()
                         .hasOnly(['name', 'category', 'city', 'price', 'photo']);

      // 只允许餐厅创建者删除餐厅（这里简化，实际可能需要管理员权限）
      allow delete: if isAuthenticated();
    }

    // 评论子集合规则
    match /restaurants/{restaurantId}/ratings/{ratingId} {
      // 允许所有已认证用户读取评论
      allow read: if isAuthenticated();

      // 只允许已认证用户创建评论
      allow create: if isAuthenticated() &&
                       request.resource.data.keys().hasAll(['userId', 'rating', 'text', 'userName', 'timestamp']) &&
                       request.resource.data.userId == request.auth.uid &&
                       request.resource.data.rating is number &&
                       request.resource.data.rating >= 1 &&
                       request.resource.data.rating <= 5;

      // 不允许更新评论（评论应该是不可变的）
      allow update: if false;

      // 只允许评论创建者删除自己的评论
      allow delete: if isAuthenticated() && resource.data.userId == request.auth.uid;
    }

    // 用户资料集合（可选，用于存储额外用户信息）
    match /users/{userId} {
      // 只允许用户读取和修改自己的资料
      allow read, write: if isAuthenticated() && request.auth.uid == userId;
    }
  }
}
```

## 创建认证相关的 UI 组件

创建 `lib/widgets/auth_gate.dart` 来控制应用访问：

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

import '../../providers/auth_provider.dart';
import '../../firebase-firestore-tutorial/auth_screen.dart';
import '../../firebase-firestore-tutorial/home_screen.dart';

/// 认证守卫组件
class AuthGate extends StatelessWidget {
  const AuthGate({super.key});

  @override
  Widget build(BuildContext context) {
    return Consumer<AuthProvider>(
      builder: (context, authProvider, _) {
        switch (authProvider.state) {
          case AuthState.initial:
          case AuthState.loading:
            return const Scaffold(
              body: Center(
                child: CircularProgressIndicator(),
              ),
            );

          case AuthState.authenticated:
            return const HomeScreen();

          case AuthState.unauthenticated:
          case AuthState.error:
            return AuthScreen(errorMessage: authProvider.errorMessage);
        }
      },
    );
  }
}
```

创建 `lib/widgets/auth_screen.dart`：

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

import '../../providers/auth_provider.dart';

class AuthScreen extends StatefulWidget {
  final String? errorMessage;

  const AuthScreen({super.key, this.errorMessage});

  @override
  State<AuthScreen> createState() => _AuthScreenState();
}

class _AuthScreenState extends State<AuthScreen> {
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  final _displayNameController = TextEditingController();

  bool _isSignUp = false;
  bool _isLoading = false;

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    _displayNameController.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    final authProvider = context.read<AuthProvider>();

    if (_emailController.text.isEmpty || _passwordController.text.isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('请填写所有必填字段')),
      );
      return;
    }

    setState(() => _isLoading = true);

    try {
      if (_isSignUp) {
        await authProvider.signUpWithEmailAndPassword(
          _emailController.text,
          _passwordController.text,
        );

        // 如果注册成功且提供了显示名称，更新用户资料
        if (_displayNameController.text.isNotEmpty) {
          await authProvider.updateProfile(displayName: _displayNameController.text);
        }
      } else {
        await authProvider.signInWithEmailAndPassword(
          _emailController.text,
          _passwordController.text,
        );
      }
    } catch (e) {
      // 错误信息会通过 AuthProvider 处理
    } finally {
      setState(() => _isLoading = false);
    }
  }

  Future<void> _signInAnonymously() async {
    setState(() => _isLoading = true);

    try {
      await context.read<AuthProvider>().signInAnonymously();
    } catch (e) {
      // 错误信息会通过 AuthProvider 处理
    } finally {
      setState(() => _isLoading = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    final authProvider = context.watch<AuthProvider>();

    return Scaffold(
      appBar: AppBar(
        title: Text(_isSignUp ? '注册' : '登录'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            if (authProvider.errorMessage != null)
              Container(
                padding: const EdgeInsets.all(8),
                color: Colors.red.shade100,
                child: Text(
                  authProvider.errorMessage!,
                  style: const TextStyle(color: Colors.red),
                ),
              ),

            const SizedBox(height: 16),

            TextField(
              controller: _emailController,
              decoration: const InputDecoration(
                labelText: '邮箱',
                border: OutlineInputBorder(),
              ),
              keyboardType: TextInputType.emailAddress,
            ),

            const SizedBox(height: 16),

            TextField(
              controller: _passwordController,
              decoration: const InputDecoration(
                labelText: '密码',
                border: OutlineInputBorder(),
              ),
              obscureText: true,
            ),

            if (_isSignUp) ...[
              const SizedBox(height: 16),
              TextField(
                controller: _displayNameController,
                decoration: const InputDecoration(
                  labelText: '显示名称',
                  border: OutlineInputBorder(),
                ),
              ),
            ],

            const SizedBox(height: 24),

            if (_isLoading)
              const CircularProgressIndicator()
            else ...[
              ElevatedButton(
                onPressed: _submit,
                child: Text(_isSignUp ? '注册' : '登录'),
              ),

              const SizedBox(height: 16),

              TextButton(
                onPressed: () => setState(() => _isSignUp = !_isSignUp),
                child: Text(_isSignUp ? '已有账户？去登录' : '没有账户？去注册'),
              ),

              const SizedBox(height: 24),

              const Text('或'),

              const SizedBox(height: 16),

              OutlinedButton(
                onPressed: _signInAnonymously,
                child: const Text('匿名访问'),
              ),
            ],
          ],
        ),
      ),
    );
  }
}
```

## 更新主应用文件

更新 `lib/main.dart` 以集成认证：

```dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

import '../../firebase-firestore-tutorial/firebase_options.dart';
import '../../firebase-firestore-tutorial/providers/auth_provider.dart';
import '../../firebase-firestore-tutorial/services/auth_service.dart';
import '../../firebase-firestore-tutorial/widgets/auth_gate.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // 初始化 Firebase
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );

  // 创建服务实例
  final authService = AuthService();
  final authProvider = AuthProvider(authService);

  runApp(MyApp(authProvider: authProvider));
}

class MyApp extends StatelessWidget {
  final AuthProvider authProvider;

  const MyApp({super.key, required this.authProvider});

  @override
  Widget build(BuildContext context) {
    return MultiProvider(
      providers: [
        ChangeNotifierProvider.value(value: authProvider),
      ],
      child: MaterialApp(
        title: 'Firestore Tutorial',
        theme: ThemeData(
          primarySwatch: Colors.blue,
        ),
        home: const AuthGate(),
      ),
    );
  }
}
```

## 实践练习

1. **实现认证服务**：创建 AuthService 类，实现各种认证方法
2. **创建认证状态管理**：实现 AuthProvider 来管理认证状态
3. **配置安全规则**：在 Firebase 控制台配置适当的安全规则
4. **创建认证界面**：实现登录、注册和匿名访问界面

## 安全注意事项

1. **密码强度**：在生产环境中验证密码强度
2. **邮箱验证**：考虑要求用户验证邮箱地址
3. **安全规则测试**：使用 Firebase 模拟器测试安全规则
4. **敏感数据**：不要在客户端存储敏感信息

## 总结与检查清单

通过本章的学习，你应该已经：

- [ ] 理解了 Firebase Authentication 的基本概念
- [ ] 实现了多种认证方式（匿名、邮箱密码）
- [ ] 创建了认证状态管理类
- [ ] 配置了适当的 Firestore 安全规则
- [ ] 创建了认证相关的用户界面
- [ ] 集成了认证守卫来控制应用访问

## 下一步

在下一章中，我们将学习如何实现实时数据流和监听器，让应用能够响应数据的实时变化。
