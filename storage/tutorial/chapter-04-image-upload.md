# 第 4 章：图片上传功能

## 图片选择

在实现上传功能之前，我们需要让用户能够选择要上传的图片。Flutter 提供了 `image_picker` 包来实现这个功能。

### 配置 ImagePicker

首先在状态类中初始化 ImagePicker：

```dart
import 'dart:io';
import 'package:image_picker/image_picker.dart';
import 'package:firebase_storage/firebase_storage.dart';

class _HomeScreenState extends State<HomeScreen> {
  final _imagePicker = ImagePicker();
  File? _image;

  // ... 其他代码
}
```

### 实现图片选择

创建选择图片的方法：

```dart
Future<void> _pickImage() async {
  try {
    final XFile? pickedFile = await _imagePicker.pickImage(
      source: ImageSource.gallery,
      maxWidth: 1920,
      maxHeight: 1080,
      imageQuality: 80,
    );

    if (pickedFile != null) {
      setState(() {
        _image = File(pickedFile.path);
      });
    }
  } catch (e) {
    print('选择图片失败: $e');
  }
}
```

### 参数说明

- **source**: 图片来源，可以是 `ImageSource.gallery`（相册）或 `ImageSource.camera`（相机）
- **maxWidth/maxHeight**: 图片最大尺寸，用于压缩大图片
- **imageQuality**: 图片质量，0-100 之间的值

## 图片预览

选择图片后，我们需要在界面上显示预览：

```dart
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(
      title: const Text('Firebase Storage'),
    ),
    body: Center(
      child: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // 图片预览
            if (_image != null)
              Container(
                height: 200,
                width: 200,
                decoration: BoxDecoration(
                  border: Border.all(color: Colors.grey),
                  borderRadius: BorderRadius.circular(8),
                ),
                child: Image.file(
                  _image!,
                  fit: BoxFit.cover,
                ),
              ),

            const SizedBox(height: 20),

            // 控制按钮
            if (_image != null) _buildActionButtons(),

            const SizedBox(height: 20),

            // 提示文字
            const Text(
              '选择一张图片上传到 Firebase Storage',
              style: TextStyle(fontSize: 16),
            ),

            const SizedBox(height: 20),

            // 选择图片按钮
            ElevatedButton.icon(
              onPressed: _image == null ? _pickImage : null,
              icon: const Icon(Icons.photo_library),
              label: const Text('选择图片'),
            ),
          ],
        ),
      ),
    ),
  );
}
```

## 上传到 Firebase Storage

### 创建存储引用

```dart
class _HomeScreenState extends State<HomeScreen> {
  final Reference _storage = FirebaseStorage.instance.ref();

  // ... 其他代码
}
```

### 实现上传功能

```dart
Future<void> _uploadImage() async {
  if (_image == null) return;

  try {
    // 从文件路径提取文件名
    final String fileName = _image!.path.split('/').last;

    // 创建存储引用
    final Reference imageRef = _storage.child('images/$fileName');

    // 开始上传
    final UploadTask uploadTask = imageRef.putFile(_image!);

    // 可选：监听上传进度
    uploadTask.snapshotEvents.listen((TaskSnapshot snapshot) {
      final double progress = (snapshot.bytesTransferred / snapshot.totalBytes) * 100;
      print('上传进度: $progress%');
    });

    // 等待上传完成
    final TaskSnapshot snapshot = await uploadTask;

    // 获取下载 URL（可选）
    final String downloadURL = await snapshot.ref.getDownloadURL();

    print('上传成功! 下载 URL: $downloadURL');

    // 显示成功消息
    if (mounted) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('图片上传成功!')),
      );
    }
  } on FirebaseException catch (e) {
    print('上传失败: $e');
    if (mounted) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('上传失败: ${e.message}')),
      );
    }
  }
}
```

### 上传步骤详解

1. **验证文件**: 确保 `_image` 不为空
2. **生成文件名**: 从文件路径提取或生成唯一文件名
3. **创建引用**: 使用 `child()` 方法指定存储路径
4. **执行上传**: 调用 `putFile()` 开始上传
5. **处理结果**: 获取下载 URL 或处理错误

## 完整的 HomeScreen 实现

```dart
import 'dart:io';
import 'package:firebase_storage/firebase_storage.dart';
import 'package:flutter/material.dart';
import 'package:image_picker/image_picker.dart';

class HomeScreen extends StatefulWidget {
  const HomeScreen({super.key});

  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  final Reference _storage = FirebaseStorage.instance.ref();
  final ImagePicker _imagePicker = ImagePicker();
  File? _image;

  Future<void> _pickImage() async {
    try {
      final XFile? pickedFile = await _imagePicker.pickImage(
        source: ImageSource.gallery,
        maxWidth: 1920,
        maxHeight: 1080,
        imageQuality: 80,
      );

      if (pickedFile != null) {
        setState(() {
          _image = File(pickedFile.path);
        });
      }
    } catch (e) {
      print('选择图片失败: $e');
    }
  }

  Future<void> _uploadImage() async {
    if (_image == null) return;

    try {
      final String fileName = _image!.path.split('/').last;
      final Reference imageRef = _storage.child('images/$fileName');

      final UploadTask uploadTask = imageRef.putFile(_image!);

      // 监听上传进度（可选）
      uploadTask.snapshotEvents.listen((TaskSnapshot snapshot) {
        final double progress = (snapshot.bytesTransferred / snapshot.totalBytes) * 100;
        print('上传进度: $progress%');
      });

      await uploadTask;

      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('图片上传成功!')),
        );
      }
    } on FirebaseException catch (e) {
      print('上传失败: $e');
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('上传失败: ${e.message}')),
        );
      }
    }
  }

  void _clearImage() {
    setState(() {
      _image = null;
    });
  }

  Widget _buildActionButtons() {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceEvenly,
      children: [
        ElevatedButton.icon(
          onPressed: _uploadImage,
          icon: const Icon(Icons.cloud_upload),
          label: const Text('上传到 Firebase'),
        ),
        ElevatedButton.icon(
          onPressed: _clearImage,
          icon: const Icon(Icons.clear),
          label: const Text('清除图片'),
        ),
      ],
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Firebase Storage'),
        actions: [
          IconButton(
            onPressed: () {
              Navigator.of(context).pushNamed('/library');
            },
            icon: const Icon(Icons.photo_library),
          ),
        ],
      ),
      body: Center(
        child: Padding(
          padding: const EdgeInsets.all(16.0),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              if (_image != null)
                Container(
                  height: 200,
                  width: 200,
                  decoration: BoxDecoration(
                    border: Border.all(color: Colors.grey),
                    borderRadius: BorderRadius.circular(8),
                  ),
                  child: Image.file(
                    _image!,
                    fit: BoxFit.cover,
                  ),
                ),

              const SizedBox(height: 20),

              if (_image != null) _buildActionButtons(),

              const SizedBox(height: 20),

              const Text(
                '选择一张图片上传到 Firebase Storage',
                style: TextStyle(fontSize: 16),
              ),

              const SizedBox(height: 20),

              ElevatedButton.icon(
                onPressed: _image == null ? _pickImage : null,
                icon: const Icon(Icons.photo_library),
                label: const Text('选择图片'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

## 练习

1. 添加相机拍摄功能，让用户可以直接拍照上传
2. 实现上传进度条显示
3. 添加文件类型和大小验证
4. 实现多文件批量上传功能

## 小结

本章我们实现了完整的图片上传功能：

- ✅ 使用 image_picker 选择图片
- ✅ 图片预览和界面设计
- ✅ 上传到 Firebase Storage
- ✅ 错误处理和用户反馈
- ✅ 进度监听（可选）

在下一章中，我们将学习如何从 Firebase Storage 下载和展示图片。
