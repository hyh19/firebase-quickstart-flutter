# 第 3 章：Firestore 基本 CRUD 操作

## 简介

本章将介绍 Firestore 的基本 CRUD（创建、读取、更新、删除）操作。我们将学习如何使用 Firestore SDK 执行这些操作，并实现一个数据提供者类来管理餐厅数据。

## Firestore 集合和文档

### 集合 (Collections) 和文档 (Documents)

Firestore 使用 **集合(Collections)** 和 **文档(Documents)** 的层次结构：

- **集合**：文档的容器，类似于关系数据库中的表
- **文档**：包含字段和值的记录，类似于关系数据库中的行
- **字段**：文档中的数据，支持多种数据类型

### 我们的数据结构

```text
restaurants (集合)
├── restaurant_id_1 (文档)
│   ├── name: "意大利厨房"
│   ├── category: "意大利菜"
│   └── ...
│   └── ratings (子集合)
│       ├── rating_id_1 (文档)
│       ├── rating_id_2 (文档)
│       └── ...
├── restaurant_id_2 (文档)
└── ...
```

## 创建数据提供者接口

首先创建抽象接口来定义数据操作：

```dart
import '../../models/restaurant.dart';
import '../../models/review.dart';
import '../../models/filter.dart';

/// 餐厅数据提供者接口
abstract class RestaurantProvider {
  /// 获取所有餐厅的流
  Stream<List<Restaurant>> get allRestaurants;

  /// 添加餐厅
  Future<void> addRestaurant(Restaurant restaurant);

  /// 批量添加餐厅
  Future<void> addRestaurants(List<Restaurant> restaurants);

  /// 删除餐厅
  Future<void> deleteRestaurant(String restaurantId);

  /// 更新餐厅信息
  Future<void> updateRestaurant(String restaurantId, Map<String, dynamic> updates);

  /// 根据ID获取餐厅
  Future<Restaurant?> getRestaurantById(String restaurantId);

  /// 加载所有餐厅
  void loadAllRestaurants();

  /// 加载筛选后的餐厅
  void loadFilteredRestaurants(Filter filter);

  /// 添加评论
  Future<void> addReview({required String restaurantId, required Review review});

  /// 获取餐厅的所有评论
  Future<List<Review>> getReviewsForRestaurant(String restaurantId);

  /// 释放资源
  void dispose();
}
```

## 实现 Firestore 数据提供者

创建 `lib/providers/firestore_restaurant_provider.dart`：

```dart
import 'dart:async';

import 'package:cloud_firestore/cloud_firestore.dart';

import '../../models/filter.dart';
import '../../models/restaurant.dart';
import '../../models/review.dart';
import '../../firebase-firestore-tutorial/restaurant_provider.dart';

/// Firestore 实现的餐厅数据提供者
class FirestoreRestaurantProvider implements RestaurantProvider {
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  final StreamController<List<Restaurant>> _restaurantsController =
      StreamController<List<Restaurant>>.broadcast();

  @override
  late final Stream<List<Restaurant>> allRestaurants;

  FirestoreRestaurantProvider() {
    allRestaurants = _restaurantsController.stream;
  }

  @override
  Future<void> addRestaurant(Restaurant restaurant) async {
    try {
      // 添加到 restaurants 集合
      final docRef = await _firestore.collection('restaurants').add(restaurant.toMap());
      print('餐厅已添加，ID: ${docRef.id}');
    } catch (e) {
      print('添加餐厅失败: $e');
      rethrow;
    }
  }

  @override
  Future<void> addRestaurants(List<Restaurant> restaurants) async {
    // 批量添加餐厅
    for (final restaurant in restaurants) {
      await addRestaurant(restaurant);
    }
  }

  @override
  Future<void> deleteRestaurant(String restaurantId) async {
    try {
      await _firestore.collection('restaurants').doc(restaurantId).delete();
      print('餐厅已删除，ID: $restaurantId');
    } catch (e) {
      print('删除餐厅失败: $e');
      rethrow;
    }
  }

  @override
  Future<void> updateRestaurant(String restaurantId, Map<String, dynamic> updates) async {
    try {
      await _firestore.collection('restaurants').doc(restaurantId).update(updates);
      print('餐厅已更新，ID: $restaurantId');
    } catch (e) {
      print('更新餐厅失败: $e');
      rethrow;
    }
  }

  @override
  Future<Restaurant?> getRestaurantById(String restaurantId) async {
    try {
      final doc = await _firestore.collection('restaurants').doc(restaurantId).get();
      if (doc.exists) {
        return Restaurant.fromSnapshot(doc);
      }
      return null;
    } catch (e) {
      print('获取餐厅失败: $e');
      return null;
    }
  }

  @override
  void loadAllRestaurants() {
    try {
      final query = _firestore
          .collection('restaurants')
          .orderBy('avgRating', descending: true)
          .limit(50);

      query.snapshots().listen(
        (snapshot) {
          final restaurants = snapshot.docs
              .map((doc) => Restaurant.fromSnapshot(doc))
              .toList();
          _restaurantsController.add(restaurants);
        },
        onError: (error) {
          print('加载餐厅列表失败: $error');
          _restaurantsController.addError(error);
        },
      );
    } catch (e) {
      print('设置餐厅监听器失败: $e');
    }
  }

  @override
  void loadFilteredRestaurants(Filter filter) {
    try {
      Query query = _firestore.collection('restaurants');

      // 应用筛选条件
      if (filter.category != null) {
        query = query.where('category', isEqualTo: filter.category);
      }
      if (filter.city != null) {
        query = query.where('city', isEqualTo: filter.city);
      }
      if (filter.price != null) {
        query = query.where('price', isEqualTo: filter.price);
      }

      // 排序
      final sortField = filter.sort ?? 'avgRating';
      query = query.orderBy(sortField, descending: true).limit(50);

      query.snapshots().listen(
        (snapshot) {
          final restaurants = snapshot.docs
              .map((doc) => Restaurant.fromSnapshot(doc))
              .toList();
          _restaurantsController.add(restaurants);
        },
        onError: (error) {
          print('加载筛选餐厅失败: $error');
          _restaurantsController.addError(error);
        },
      );
    } catch (e) {
      print('设置筛选监听器失败: $e');
    }
  }

  @override
  Future<void> addReview({
    required String restaurantId,
    required Review review,
  }) async {
    try {
      // 使用事务确保数据一致性
      await _firestore.runTransaction((transaction) async {
        // 获取餐厅文档
        final restaurantRef = _firestore.collection('restaurants').doc(restaurantId);
        final restaurantDoc = await transaction.get(restaurantRef);

        if (!restaurantDoc.exists) {
          throw Exception('餐厅不存在');
        }

        final restaurant = Restaurant.fromSnapshot(restaurantDoc);

        // 创建新的评论文档
        final reviewRef = restaurantRef.collection('ratings').doc();
        transaction.set(reviewRef, review.toMap());

        // 更新餐厅的评分统计
        final newNumRatings = restaurant.numRatings + 1;
        final newAvgRating = ((restaurant.avgRating * restaurant.numRatings) + review.rating) / newNumRatings;

        transaction.update(restaurantRef, {
          'numRatings': newNumRatings,
          'avgRating': newAvgRating,
        });
      });

      print('评论已添加，餐厅ID: $restaurantId');
    } catch (e) {
      print('添加评论失败: $e');
      rethrow;
    }
  }

  @override
  Future<List<Review>> getReviewsForRestaurant(String restaurantId) async {
    try {
      final snapshot = await _firestore
          .collection('restaurants')
          .doc(restaurantId)
          .collection('ratings')
          .orderBy('timestamp', descending: true)
          .get();

      return snapshot.docs
          .map((doc) => Review.fromSnapshot(doc))
          .toList();
    } catch (e) {
      print('获取评论失败: $e');
      return [];
    }
  }

  @override
  void dispose() {
    _restaurantsController.close();
  }
}
```

## 实现基本的 CRUD 服务类

创建 `lib/services/restaurant_service.dart` 来提供更高级别的操作：

```dart
import '../../models/restaurant.dart';
import '../../models/review.dart';
import '../../providers/restaurant_provider.dart';

/// 餐厅服务类，提供高级别的业务逻辑
class RestaurantService {
  final RestaurantProvider _provider;

  RestaurantService(this._provider);

  /// 创建餐厅
  Future<String?> createRestaurant(Restaurant restaurant) async {
    try {
      // 验证数据
      restaurant.validate();

      // 添加到数据库
      final docRef = await _provider.addRestaurant(restaurant);

      // 注意：这里需要修改提供者接口来返回文档ID
      // 暂时返回null，实际使用时需要修改接口
      return null;
    } catch (e) {
      print('创建餐厅失败: $e');
      return null;
    }
  }

  /// 更新餐厅评分
  Future<void> updateRestaurantRating(String restaurantId, double newRating, int newNumRatings) async {
    try {
      await _provider.updateRestaurant(restaurantId, {
        'avgRating': newRating,
        'numRatings': newNumRatings,
      });
    } catch (e) {
      print('更新餐厅评分失败: $e');
      rethrow;
    }
  }

  /// 删除餐厅及其所有评论
  Future<void> deleteRestaurantWithReviews(String restaurantId) async {
    try {
      // 首先删除所有评论
      final reviews = await _provider.getReviewsForRestaurant(restaurantId);
      // 注意：实际删除评论需要事务，这里简化处理

      // 删除餐厅
      await _provider.deleteRestaurant(restaurantId);
    } catch (e) {
      print('删除餐厅失败: $e');
      rethrow;
    }
  }

  /// 获取餐厅统计信息
  Future<Map<String, dynamic>> getRestaurantStats(String restaurantId) async {
    try {
      final restaurant = await _provider.getRestaurantById(restaurantId);
      final reviews = await _provider.getReviewsForRestaurant(restaurantId);

      if (restaurant == null) {
        throw Exception('餐厅不存在');
      }

      // 计算统计信息
      final ratingDistribution = <int, int>{};
      for (final review in reviews) {
        final rating = review.rating.round();
        ratingDistribution[rating] = (ratingDistribution[rating] ?? 0) + 1;
      }

      return {
        'totalReviews': reviews.length,
        'averageRating': restaurant.avgRating,
        'ratingDistribution': ratingDistribution,
        'positiveReviews': reviews.where((r) => r.isPositive).length,
      };
    } catch (e) {
      print('获取统计信息失败: $e');
      rethrow;
    }
  }

  /// 批量创建测试数据
  Future<void> createTestData(int count) async {
    try {
      final restaurants = List.generate(count, (_) => Restaurant.random());
      await _provider.addRestaurants(restaurants);
      print('已创建 $count 个测试餐厅');
    } catch (e) {
      print('创建测试数据失败: $e');
      rethrow;
    }
  }
}
```

## 错误处理和重试机制

创建 `lib/services/error_handler.dart` 来处理常见的 Firestore 错误：

```dart
import 'package:cloud_firestore/cloud_firestore.dart';

/// Firestore 错误处理器
class FirestoreErrorHandler {
  /// 处理 Firestore 异常并返回用户友好的错误信息
  static String getErrorMessage(Object error) {
    if (error is FirebaseException) {
      switch (error.code) {
        case 'permission-denied':
          return '权限被拒绝，请检查您的认证状态';
        case 'not-found':
          return '请求的文档不存在';
        case 'already-exists':
          return '文档已存在';
        case 'resource-exhausted':
          return '超出配额限制，请稍后再试';
        case 'failed-precondition':
          return '操作无法执行，请检查索引配置';
        case 'unavailable':
          return '服务暂时不可用，请稍后再试';
        case 'deadline-exceeded':
          return '请求超时，请检查网络连接';
        default:
          return '数据库操作失败: ${error.message}';
      }
    }

    return '未知错误: $error';
  }

  /// 判断错误是否可以重试
  static bool isRetryableError(Object error) {
    if (error is FirebaseException) {
      return error.code == 'unavailable' ||
             error.code == 'deadline-exceeded' ||
             error.code == 'resource-exhausted';
    }
    return false;
  }

  /// 执行带重试的异步操作
  static Future<T> executeWithRetry<T>(
    Future<T> Function() operation, {
    int maxRetries = 3,
    Duration delay = const Duration(seconds: 1),
  }) async {
    int attempts = 0;

    while (attempts < maxRetries) {
      try {
        return await operation();
      } catch (e) {
        attempts++;

        if (attempts >= maxRetries || !isRetryableError(e)) {
          throw e;
        }

        print('操作失败，将在 ${delay.inSeconds} 秒后重试 ($attempts/$maxRetries): $e');
        await Future.delayed(delay);

        // 指数退避
        delay *= 2;
      }
    }

    throw Exception('重试次数超过限制');
  }
}
```

## 使用示例

创建 `lib/widgets/crud_demo.dart` 来演示 CRUD 操作：

```dart
import 'package:flutter/material.dart';

import '../../models/restaurant.dart';
import '../../providers/firestore_restaurant_provider.dart';
import '../../services/error_handler.dart';

class CrudDemo extends StatefulWidget {
  const CrudDemo({super.key});

  @override
  State<CrudDemo> createState() => _CrudDemoState();
}

class _CrudDemoState extends State<CrudDemo> {
  final FirestoreRestaurantProvider _provider = FirestoreRestaurantProvider();
  final TextEditingController _nameController = TextEditingController();

  @override
  void initState() {
    super.initState();
    _provider.loadAllRestaurants();
  }

  @override
  void dispose() {
    _provider.dispose();
    _nameController.dispose();
    super.dispose();
  }

  Future<void> _createRestaurant() async {
    if (_nameController.text.isEmpty) return;

    try {
      final restaurant = Restaurant._(
        name: _nameController.text,
        category: '测试类别',
        city: '测试城市',
      );

      await FirestoreErrorHandler.executeWithRetry(
        () => _provider.addRestaurant(restaurant),
      );

      _nameController.clear();
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('餐厅创建成功')),
      );
    } catch (e) {
      final errorMessage = FirestoreErrorHandler.getErrorMessage(e);
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('创建失败: $errorMessage')),
      );
    }
  }

  Future<void> _createTestData() async {
    try {
      await FirestoreErrorHandler.executeWithRetry(
        () => _provider.addRestaurants(
          List.generate(5, (_) => Restaurant.random()),
        ),
      );

      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('测试数据创建成功')),
      );
    } catch (e) {
      final errorMessage = FirestoreErrorHandler.getErrorMessage(e);
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('创建测试数据失败: $errorMessage')),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('CRUD 操作演示'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            // 创建餐厅表单
            TextField(
              controller: _nameController,
              decoration: const InputDecoration(
                labelText: '餐厅名称',
                border: OutlineInputBorder(),
              ),
            ),
            const SizedBox(height: 16),
            Row(
              children: [
                ElevatedButton(
                  onPressed: _createRestaurant,
                  child: const Text('创建餐厅'),
                ),
                const SizedBox(width: 16),
                ElevatedButton(
                  onPressed: _createTestData,
                  child: const Text('创建测试数据'),
                ),
              ],
            ),

            const SizedBox(height: 24),
            const Text(
              '餐厅列表',
              style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
            ),

            // 餐厅列表
            Expanded(
              child: StreamBuilder<List<Restaurant>>(
                stream: _provider.allRestaurants,
                builder: (context, snapshot) {
                  if (snapshot.hasError) {
                    return Center(
                      child: Text('错误: ${snapshot.error}'),
                    );
                  }

                  if (!snapshot.hasData) {
                    return const Center(child: CircularProgressIndicator());
                  }

                  final restaurants = snapshot.data!;

                  return ListView.builder(
                    itemCount: restaurants.length,
                    itemBuilder: (context, index) {
                      final restaurant = restaurants[index];
                      return ListTile(
                        title: Text(restaurant.name),
                        subtitle: Text(
                          '${restaurant.category ?? '未分类'} • ${restaurant.city ?? '未设置城市'}',
                        ),
                        trailing: Text('⭐ ${restaurant.avgRating.toStringAsFixed(1)}'),
                      );
                    },
                  );
                },
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

## 实践练习

1. **实现 CRUD 操作**：创建 FirestoreRestaurantProvider 类，实现所有 CRUD 方法
2. **错误处理**：添加适当的错误处理和用户友好的错误信息
3. **测试功能**：创建一个测试页面来验证所有 CRUD 操作是否正常工作

```dart
// 测试代码示例
void testCrudOperations() async {
  final provider = FirestoreRestaurantProvider();

  try {
    // 创建餐厅
    final restaurant = Restaurant.random();
    await provider.addRestaurant(restaurant);
    print('餐厅创建成功');

    // 读取餐厅
    provider.loadAllRestaurants();
    print('餐厅列表加载成功');

    // 更新餐厅（如果有ID的话）
    // await provider.updateRestaurant(restaurantId, {'name': '新名称'});

    // 删除餐厅（如果有ID的话）
    // await provider.deleteRestaurant(restaurantId);

  } catch (e) {
    print('CRUD 操作失败: $e');
  } finally {
    provider.dispose();
  }
}
```

## 总结与检查清单

通过本章的学习，你应该已经：

- [ ] 理解了 Firestore 的集合和文档结构
- [ ] 实现了 RestaurantProvider 接口
- [ ] 掌握了基本的 CRUD 操作（创建、读取、更新、删除）
- [ ] 学会了使用事务来确保数据一致性
- [ ] 添加了适当的错误处理和重试机制
- [ ] 创建了演示 CRUD 操作的用户界面

## 下一步

在下一章中，我们将学习如何集成 Firebase Authentication，为应用添加用户认证功能。
