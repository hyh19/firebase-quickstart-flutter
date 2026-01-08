# 第 5 章：图片下载和展示

## Storage 引用和文件列表

要下载文件，首先需要了解如何访问 Firebase Storage 中的文件和文件夹。

### 创建存储引用

```dart
class _LibraryPageState extends State<LibraryPage> {
  final Reference storageRef = FirebaseStorage.instance.ref();
  List<Uint8List>? _images;

  // ... 其他代码
}
```

### 列出文件夹中的文件

Firebase Storage 提供了 `listAll()` 方法来获取文件夹中的所有文件：

```dart
Future<void> _loadImages() async {
  try {
    // 获取 images 文件夹的引用
    final imagesRef = storageRef.child("images");

    // 列出该文件夹中的所有文件
    final ListResult result = await imagesRef.listAll();

    // 获取所有文件的引用
    final List<Reference> files = result.items;

    print('找到 ${files.length} 个文件');

    // 下载所有图片数据
    final List<Uint8List> images = [];
    for (final ref in files) {
      try {
        final Uint8List? data = await ref.getData();
        if (data != null) {
          images.add(data);
        }
      } catch (e) {
        print('下载文件失败: $e');
      }
    }

    setState(() {
      _images = images;
    });
  } on FirebaseException catch (e) {
    print('加载图片列表失败: $e');
  }
}
```

### ListAll 方法详解

`listAll()` 返回一个 `ListResult` 对象，包含：

- **items**: 文件引用列表
- **prefixes**: 子文件夹引用列表

```dart
final ListResult result = await imagesRef.listAll();

// 遍历所有文件
for (final Reference ref in result.items) {
  print('文件: ${ref.name}');
}

// 遍历所有子文件夹
for (final Reference ref in result.prefixes) {
  print('文件夹: ${ref.name}');
}
```

## 下载文件数据

Firebase Storage 提供了多种下载文件的方法：

### 1. 下载为字节数组

最直接的方法是使用 `getData()` 下载文件的原始字节数据：

```dart
final Uint8List? data = await reference.getData();
```

### 2. 获取下载 URL

获取文件的公开下载 URL，然后使用 HTTP 客户端下载：

```dart
final String downloadURL = await reference.getDownloadURL();
// 使用 http 包或 Image.network() 加载
```

### 3. 流式下载

对于大文件，可以使用流式下载来节省内存：

```dart
final Uint8List? data = await reference.getData(10 * 1024 * 1024); // 限制最大大小
```

## 图片展示

下载图片数据后，我们需要将其显示在 Flutter 界面中。

### 使用 Image.memory

`Image.memory` 可以直接从字节数组创建图片组件：

```dart
Image.memory(
  imageData,
  width: 150,
  height: 150,
  fit: BoxFit.cover,
)
```

### 创建图片网格

使用 `Wrap` 组件创建一个响应式的图片网格：

```dart
Widget _buildImageGrid() {
  if (_images == null) {
    return const CircularProgressIndicator();
  }

  return Wrap(
    spacing: 8.0,      // 水平间距
    runSpacing: 8.0,   // 垂直间距
    children: [
      for (final imageData in _images!)
        Container(
          width: 150,
          height: 150,
          decoration: BoxDecoration(
            border: Border.all(color: Colors.grey.shade300),
            borderRadius: BorderRadius.circular(8),
          ),
          child: ClipRRect(
            borderRadius: BorderRadius.circular(8),
            child: Image.memory(
              imageData,
              fit: BoxFit.cover,
            ),
          ),
        ),
    ],
  );
}
```

## 完整的 LibraryPage 实现

```dart
import 'dart:typed_data';
import 'package:firebase_storage/firebase_storage.dart';
import 'package:flutter/material.dart';

class LibraryPage extends StatefulWidget {
  const LibraryPage({super.key});

  @override
  State<LibraryPage> createState() => _LibraryPageState();
}

class _LibraryPageState extends State<LibraryPage> {
  final Reference storageRef = FirebaseStorage.instance.ref();
  List<Uint8List>? _images;
  bool _isLoading = true;

  @override
  void initState() {
    super.initState();
    _loadImages();
  }

  Future<void> _loadImages() async {
    setState(() {
      _isLoading = true;
    });

    try {
      final imagesRef = storageRef.child("images");
      final ListResult result = await imagesRef.listAll();

      final List<Uint8List> images = [];
      for (final ref in result.items) {
        try {
          final Uint8List? data = await ref.getData(5 * 1024 * 1024); // 5MB 限制
          if (data != null) {
            images.add(data);
          }
        } catch (e) {
          print('下载文件失败 ${ref.name}: $e');
        }
      }

      setState(() {
        _images = images;
        _isLoading = false;
      });
    } on FirebaseException catch (e) {
      print('加载图片失败: $e');
      setState(() {
        _isLoading = false;
      });

      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('加载图片失败: ${e.message}')),
        );
      }
    }
  }

  Widget _buildImageGrid() {
    if (_isLoading) {
      return const Center(
        child: CircularProgressIndicator(),
      );
    }

    if (_images == null || _images!.isEmpty) {
      return const Center(
        child: Text('暂无图片'),
      );
    }

    return SingleChildScrollView(
      padding: const EdgeInsets.all(16.0),
      child: Wrap(
        spacing: 12.0,
        runSpacing: 12.0,
        children: [
          for (final imageData in _images!)
            Container(
              width: 140,
              height: 140,
              decoration: BoxDecoration(
                border: Border.all(color: Colors.grey.shade300),
                borderRadius: BorderRadius.circular(12),
                boxShadow: [
                  BoxShadow(
                    color: Colors.black.withOpacity(0.1),
                    blurRadius: 4,
                    offset: const Offset(0, 2),
                  ),
                ],
              ),
              child: ClipRRect(
                borderRadius: BorderRadius.circular(12),
                child: Image.memory(
                  imageData,
                  fit: BoxFit.cover,
                ),
              ),
            ),
        ],
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text("图片库"),
        actions: [
          IconButton(
            onPressed: _loadImages,
            icon: const Icon(Icons.refresh),
          ),
        ],
      ),
      body: _buildImageGrid(),
    );
  }
}
```

## 优化和缓存

### 内存优化

对于大量图片，考虑以下优化：

1. **分页加载**: 不要一次性加载所有图片
2. **缩略图**: 先加载小图，点击后加载大图
3. **内存管理**: 及时释放不需要的图片数据

### 缓存策略

```dart
// 使用 Firebase Storage 的缓存
final Uint8List? data = await reference.getData();

// 或者使用第三方缓存库
// cached_network_image, flutter_cache_manager 等
```

### 错误处理

```dart
try {
  final data = await reference.getData(maxSize: 10 * 1024 * 1024);
  if (data != null) {
    // 处理图片数据
  }
} on FirebaseException catch (e) {
  switch (e.code) {
    case 'object-not-found':
      print('文件不存在');
      break;
    case 'unauthorized':
      print('没有访问权限');
      break;
    case 'cancelled':
      print('下载被取消');
      break;
    default:
      print('下载失败: ${e.message}');
  }
}
```

## 练习

1. 实现下拉刷新功能
2. 添加图片删除功能
3. 实现图片预览（点击放大）
4. 添加分页加载支持
5. 实现图片缓存机制

## 小结

本章我们学习了如何从 Firebase Storage 下载和展示文件：

- ✅ 使用 listAll() 列出存储文件
- ✅ 下载文件数据为字节数组
- ✅ 使用 Image.memory 展示图片
- ✅ 创建响应式图片网格布局
- ✅ 错误处理和用户反馈
- ✅ 性能优化建议

在最后一章中，我们将学习错误处理和最佳实践。
