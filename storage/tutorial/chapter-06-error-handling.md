# 第 6 章：错误处理和最佳实践

## Firebase 异常处理

Firebase Storage 操作可能会遇到各种错误，正确处理这些异常对于提供良好的用户体验至关重要。

### FirebaseException 详解

Firebase Storage 的所有异常都继承自 `FirebaseException`，包含以下属性：

- **code**: 错误代码字符串
- **message**: 人类可读的错误描述
- **plugin**: 插件名称（通常是 'firebase_storage'）

### 常见错误类型

#### 上传错误

```dart
try {
  await storageRef.putFile(file);
} on FirebaseException catch (e) {
  switch (e.code) {
    case 'unauthorized':
      print('没有上传权限');
      break;
    case 'canceled':
      print('上传被取消');
      break;
    case 'unknown':
      print('未知错误: ${e.message}');
      break;
    default:
      print('上传失败: ${e.code}');
  }
}
```

#### 下载错误

```dart
try {
  final data = await storageRef.getData();
} on FirebaseException catch (e) {
  switch (e.code) {
    case 'object-not-found':
      print('文件不存在');
      break;
    case 'unauthorized':
      print('没有下载权限');
      break;
    case 'download-size-exceeded':
      print('文件太大，无法下载');
      break;
    case 'canceled':
      print('下载被取消');
      break;
    default:
      print('下载失败: ${e.message}');
  }
}
```

#### 列表错误

```dart
try {
  await storageRef.listAll();
} on FirebaseException catch (e) {
  switch (e.code) {
    case 'unauthorized':
      print('没有列出文件的权限');
      break;
    case 'object-not-found':
      print('文件夹不存在');
      break;
    default:
      print('获取文件列表失败: ${e.message}');
  }
}
```

## 用户友好的错误提示

将技术错误转换为用户友好的提示信息：

```dart
class StorageErrorHandler {
  static String getUserFriendlyMessage(FirebaseException e) {
    switch (e.code) {
      case 'unauthorized':
        return '您没有权限执行此操作，请检查网络连接或重新登录';
      case 'object-not-found':
        return '文件不存在，可能已被删除';
      case 'quota-exceeded':
        return '存储空间不足，请清理一些文件';
      case 'download-size-exceeded':
        return '文件太大，无法下载';
      case 'canceled':
        return '操作已被取消';
      case 'invalid-argument':
        return '输入参数无效，请重试';
      default:
        return '操作失败，请稍后重试';
    }
  }
}
```

## 重试机制

对于网络相关错误，实施重试机制：

```dart
class RetryableStorage {
  static Future<T> executeWithRetry<T>(
    Future<T> Function() operation, {
    int maxAttempts = 3,
    Duration delay = const Duration(seconds: 1),
  }) async {
    int attempts = 0;

    while (attempts < maxAttempts) {
      try {
        return await operation();
      } on FirebaseException catch (e) {
        attempts++;

        // 某些错误不需要重试
        if (!_isRetryableError(e.code) || attempts >= maxAttempts) {
          rethrow;
        }

        // 等待后重试
        await Future.delayed(delay * attempts);
      }
    }

    throw Exception('超出最大重试次数');
  }

  static bool _isRetryableError(String code) {
    return [
      'unavailable',
      'deadline-exceeded',
      'internal',
      'unknown',
    ].contains(code);
  }
}
```

## 性能优化

### 1. 文件压缩

在上传前压缩图片以减少存储成本和传输时间：

```dart
import 'package:image/image.dart' as img;

Future<File> compressImage(File file) async {
  final bytes = await file.readAsBytes();
  final image = img.decodeImage(bytes);

  if (image == null) return file;

  // 压缩图片
  final compressed = img.encodeJpg(image, quality: 80);

  // 保存到临时文件
  final tempDir = await getTemporaryDirectory();
  final tempFile = File('${tempDir.path}/compressed_${file.path.split('/').last}');
  await tempFile.writeAsBytes(compressed);

  return tempFile;
}
```

### 2. 分块上传

对于大文件，使用分块上传：

```dart
Future<void> uploadLargeFile(Reference ref, File file) async {
  final chunkSize = 256 * 1024; // 256KB chunks
  final totalSize = await file.length();
  var uploadedBytes = 0;

  final uploadTask = ref.putFile(
    file,
    SettableMetadata(
      customMetadata: {'totalSize': totalSize.toString()},
    ),
  );

  // 监听上传进度
  uploadTask.snapshotEvents.listen((snapshot) {
    final progress = (snapshot.bytesTransferred / snapshot.totalBytes * 100);
    print('上传进度: $progress%');
  });

  await uploadTask;
}
```

### 3. 缓存策略

实现本地缓存以减少重复下载：

```dart
class ImageCache {
  static final Map<String, Uint8List> _cache = {};

  static Future<Uint8List?> getCachedImage(String imagePath) async {
    if (_cache.containsKey(imagePath)) {
      return _cache[imagePath];
    }

    try {
      final ref = FirebaseStorage.instance.ref().child(imagePath);
      final data = await ref.getData();
      if (data != null) {
        _cache[imagePath] = data;
      }
      return data;
    } catch (e) {
      return null;
    }
  }

  static void clearCache() {
    _cache.clear();
  }
}
```

## 安全最佳实践

### 1. 文件验证

在上传前验证文件类型和大小：

```dart
class FileValidator {
  static bool isValidImageFile(File file) {
    final validExtensions = ['.jpg', '.jpeg', '.png', '.gif', '.webp'];
    final fileName = file.path.toLowerCase();

    return validExtensions.any((ext) => fileName.endsWith(ext));
  }

  static bool isValidFileSize(File file, {int maxSizeMB = 10}) {
    final maxSizeBytes = maxSizeMB * 1024 * 1024;
    return file.lengthSync() <= maxSizeBytes;
  }

  static Future<bool> validateAndCompress(File file) async {
    if (!isValidImageFile(file)) return false;
    if (!isValidFileSize(file)) return false;

    // 压缩大图片
    if (file.lengthSync() > 1024 * 1024) { // 1MB
      final compressed = await compressImage(file);
      // 替换原文件
      await file.writeAsBytes(await compressed.readAsBytes());
    }

    return true;
  }
}
```

### 2. 安全的文件命名

生成安全的唯一文件名：

```dart
class FileNameGenerator {
  static String generateUniqueFileName(String originalName) {
    final extension = originalName.split('.').last;
    final timestamp = DateTime.now().millisecondsSinceEpoch;
    final random = Random().nextInt(10000);
    return '${timestamp}_$random.$extension';
  }

  static String sanitizeFileName(String fileName) {
    // 移除或替换不安全的字符
    return fileName
        .replaceAll(RegExp(r'[^\w\.-]'), '_')
        .toLowerCase();
  }
}
```

## 监控和调试

### 添加日志

```dart
class StorageLogger {
  static void logOperation(String operation, String path, [dynamic data]) {
    print('[$operation] $path ${data ?? ''}');
  }

  static void logError(String operation, String path, FirebaseException e) {
    print('[$operation ERROR] $path - ${e.code}: ${e.message}');
  }
}
```

### 性能监控

```dart
class StorageMetrics {
  static final Map<String, int> uploadCounts = {};
  static final Map<String, Duration> uploadTimes = {};

  static void recordUpload(String path, Duration duration) {
    uploadCounts[path] = (uploadCounts[path] ?? 0) + 1;
    uploadTimes[path] = duration;

    print('上传统计 - $path: 次数=${uploadCounts[path]}, 时间=${duration.inMilliseconds}ms');
  }
}
```

## 完整的错误处理服务

```dart
class StorageService {
  final FirebaseStorage _storage = FirebaseStorage.instance;

  Future<String?> uploadImage(File imageFile) async {
    try {
      // 验证文件
      if (!await FileValidator.validateAndCompress(imageFile)) {
        throw Exception('文件验证失败');
      }

      // 生成安全文件名
      final fileName = FileNameGenerator.generateUniqueFileName(imageFile.path.split('/').last);
      final ref = _storage.ref().child('images/$fileName');

      // 上传文件
      final startTime = DateTime.now();
      await ref.putFile(imageFile);
      final duration = DateTime.now().difference(startTime);

      // 记录指标
      StorageMetrics.recordUpload(fileName, duration);

      // 返回下载 URL
      return await ref.getDownloadURL();
    } on FirebaseException catch (e) {
      StorageLogger.logError('UPLOAD', imageFile.path, e);
      throw Exception(StorageErrorHandler.getUserFriendlyMessage(e));
    } catch (e) {
      StorageLogger.logOperation('UPLOAD_ERROR', imageFile.path, e);
      throw Exception('上传失败: ${e.toString()}');
    }
  }

  Future<List<Uint8List>> downloadImages() async {
    try {
      final imagesRef = _storage.ref().child('images');
      final result = await imagesRef.listAll();

      final images = <Uint8List>[];
      for (final ref in result.items) {
        final data = await ref.getData();
        if (data != null) {
          images.add(data);
        }
      }

      return images;
    } on FirebaseException catch (e) {
      StorageLogger.logError('DOWNLOAD', 'images/', e);
      throw Exception(StorageErrorHandler.getUserFriendlyMessage(e));
    }
  }
}
```

## 总结

通过本教程，我们学习了如何构建一个完整的 Firebase Storage 图片上传下载应用，并掌握了错误处理和性能优化的最佳实践。

### 核心要点回顾

1. **错误处理**: 正确捕获和处理 FirebaseException
2. **用户体验**: 提供友好的错误提示和加载状态
3. **性能优化**: 文件压缩、分块上传、缓存策略
4. **安全性**: 文件验证、安全命名、权限控制
5. **监控调试**: 日志记录、性能指标、错误追踪

### 进一步学习建议

- 探索 Firebase Storage 的安全规则配置
- 学习如何与 Cloud Functions 集成进行图片处理
- 研究离线支持和数据同步机制
- 深入了解 Firebase 生态系统的其他服务

现在你已经具备了使用 Firebase Storage 构建健壮应用的完整知识。开始构建你自己的文件存储解决方案吧！
