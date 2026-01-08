# 第 7 章：评论子集合与事务

## 简介

本章将介绍如何使用 Firestore 的子集合和事务功能。我们将学习如何管理餐厅评论、处理并发更新，以及确保数据一致性。

## 子集合概念

### 什么是子集合

子集合是嵌套在文档中的集合，用于表示一对多的关系：

```text
restaurants (集合)
├── restaurant_1 (文档)
│   ├── ratings (子集合)
│   │   ├── rating_1 (文档)
│   │   ├── rating_2 (文档)
│   │   └── ...
└── restaurant_2 (文档)
    └── ratings (子集合)
        └── ...
```

### 子集合 vs 嵌套数组

- **子集合**：适合大量数据、需要独立查询的场景
- **嵌套数组**：适合少量固定数据、不需要复杂查询的场景

## 实现评论系统

### 评论数据模型

```dart
class Review {
  final String? id;
  final String restaurantId;
  final String userId;
  final String userName;
  final double rating;
  final String text;
  final DateTime timestamp;

  const Review({
    this.id,
    required this.restaurantId,
    required this.userId,
    required this.userName,
    required this.rating,
    required this.text,
    required this.timestamp,
  });

  Map<String, dynamic> toMap() {
    return {
      'restaurantId': restaurantId,
      'userId': userId,
      'userName': userName,
      'rating': rating,
      'text': text,
      'timestamp': timestamp.toUtc(),
    };
  }

  factory Review.fromMap(Map<String, dynamic> map, String id) {
    return Review(
      id: id,
      restaurantId: map['restaurantId'],
      userId: map['userId'],
      userName: map['userName'],
      rating: map['rating']?.toDouble() ?? 0.0,
      text: map['text'] ?? '',
      timestamp: (map['timestamp'] as Timestamp?)?.toDate() ?? DateTime.now(),
    );
  }
}
```

### 评论服务

```dart
class ReviewService {
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  /// 添加评论并更新餐厅评分
  Future<void> addReview(Review review) async {
    await _firestore.runTransaction((transaction) async {
      // 1. 添加评论到子集合
      final reviewRef = _firestore
          .collection('restaurants')
          .doc(review.restaurantId)
          .collection('ratings')
          .doc();

      transaction.set(reviewRef, review.toMap());

      // 2. 获取当前餐厅数据
      final restaurantRef = _firestore.collection('restaurants').doc(review.restaurantId);
      final restaurantDoc = await transaction.get(restaurantRef);

      if (!restaurantDoc.exists) {
        throw Exception('餐厅不存在');
      }

      final data = restaurantDoc.data()!;
      final currentRating = (data['avgRating'] ?? 0).toDouble();
      final currentCount = (data['numRatings'] ?? 0) as int;

      // 3. 计算新的平均评分
      final newCount = currentCount + 1;
      final newRating = ((currentRating * currentCount) + review.rating) / newCount;

      // 4. 更新餐厅文档
      transaction.update(restaurantRef, {
        'avgRating': newRating,
        'numRatings': newCount,
      });
    });
  }

  /// 获取餐厅的所有评论
  Stream<List<Review>> getReviewsForRestaurant(String restaurantId) {
    return _firestore
        .collection('restaurants')
        .doc(restaurantId)
        .collection('ratings')
        .orderBy('timestamp', descending: true)
        .snapshots()
        .map((snapshot) =>
            snapshot.docs.map((doc) => Review.fromMap(doc.data(), doc.id)).toList());
  }

  /// 删除评论并更新餐厅评分
  Future<void> deleteReview(String restaurantId, String reviewId) async {
    await _firestore.runTransaction((transaction) async {
      // 1. 获取要删除的评论
      final reviewRef = _firestore
          .collection('restaurants')
          .doc(restaurantId)
          .collection('ratings')
          .doc(reviewId);

      final reviewDoc = await transaction.get(reviewRef);
      if (!reviewDoc.exists) {
        throw Exception('评论不存在');
      }

      final reviewData = reviewDoc.data()!;
      final reviewRating = (reviewData['rating'] ?? 0).toDouble();

      // 2. 删除评论
      transaction.delete(reviewRef);

      // 3. 更新餐厅评分
      final restaurantRef = _firestore.collection('restaurants').doc(restaurantId);
      final restaurantDoc = await transaction.get(restaurantRef);

      if (restaurantDoc.exists) {
        final data = restaurantDoc.data()!;
        final currentRating = (data['avgRating'] ?? 0).toDouble();
        final currentCount = (data['numRatings'] ?? 0) as int;

        if (currentCount > 1) {
          // 计算新平均分：(总分 - 删除的分数) / (总数 - 1)
          final newRating = ((currentRating * currentCount) - reviewRating) / (currentCount - 1);
          transaction.update(restaurantRef, {
            'avgRating': newRating,
            'numRatings': currentCount - 1,
          });
        } else {
          // 如果这是最后一个评论，重置为0
          transaction.update(restaurantRef, {
            'avgRating': 0.0,
            'numRatings': 0,
          });
        }
      }
    });
  }

  /// 获取用户的评论
  Future<List<Review>> getUserReviews(String userId) async {
    final querySnapshot = await _firestore
        .collectionGroup('ratings')
        .where('userId', isEqualTo: userId)
        .orderBy('timestamp', descending: true)
        .get();

    return querySnapshot.docs
        .map((doc) => Review.fromMap(doc.data(), doc.id))
        .toList();
  }

  /// 更新评论
  Future<void> updateReview(String restaurantId, String reviewId, String newText, double newRating) async {
    await _firestore.runTransaction((transaction) async {
      // 1. 获取原始评论
      final reviewRef = _firestore
          .collection('restaurants')
          .doc(restaurantId)
          .collection('ratings')
          .doc(reviewId);

      final reviewDoc = await transaction.get(reviewRef);
      if (!reviewDoc.exists) {
        throw Exception('评论不存在');
      }

      final oldData = reviewDoc.data()!;
      final oldRating = (oldData['rating'] ?? 0).toDouble();

      // 2. 更新评论
      transaction.update(reviewRef, {
        'text': newText,
        'rating': newRating,
        'timestamp': FieldValue.serverTimestamp(),
      });

      // 3. 更新餐厅平均分
      final restaurantRef = _firestore.collection('restaurants').doc(restaurantId);
      final restaurantDoc = await transaction.get(restaurantRef);

      if (restaurantDoc.exists) {
        final data = restaurantDoc.data()!;
        final currentRating = (data['avgRating'] ?? 0).toDouble();
        final currentCount = (data['numRatings'] ?? 0) as int;

        // 计算新平均分：旧平均分 + (新分数 - 旧分数) / 总数
        final ratingDiff = newRating - oldRating;
        final newAvgRating = currentRating + (ratingDiff / currentCount);

        transaction.update(restaurantRef, {
          'avgRating': newAvgRating,
        });
      }
    });
  }
}
```

## 事务最佳实践

### 事务设计原则

1. **原子性**：事务中的所有操作要么全部成功，要么全部失败
2. **一致性**：事务完成后数据处于一致状态
3. **隔离性**：并发事务不会相互干扰
4. **持久性**：事务提交后数据永久保存

### 事务使用场景

1. **计数器更新**：增加评论数、更新平均分
2. **库存管理**：扣减库存、记录订单
3. **转账操作**：从一个账户转到另一个账户
4. **批量更新**：需要保证多文档一致性的操作

### 事务限制

- 事务最多只能读取/写入 500 个文档
- 事务必须在 270 秒内完成
- 事务中的读取操作必须在写入操作之前

## 批量操作

### 批量写入

```dart
class BatchOperations {
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  /// 批量添加评论
  Future<void> addMultipleReviews(String restaurantId, List<Review> reviews) async {
    final batch = _firestore.batch();

    // 添加所有评论
    for (final review in reviews) {
      final reviewRef = _firestore
          .collection('restaurants')
          .doc(restaurantId)
          .collection('ratings')
          .doc();
      batch.set(reviewRef, review.toMap());
    }

    // 计算新的平均评分
    final totalRating = reviews.fold<double>(0, (sum, review) => sum + review.rating);
    final averageRating = totalRating / reviews.length;

    // 更新餐厅
    final restaurantRef = _firestore.collection('restaurants').doc(restaurantId);
    batch.update(restaurantRef, {
      'avgRating': averageRating,
      'numRatings': FieldValue.increment(reviews.length),
    });

    await batch.commit();
  }

  /// 批量删除用户的所有评论
  Future<void> deleteAllUserReviews(String userId) async {
    // 首先找到用户的所有评论
    final querySnapshot = await _firestore
        .collectionGroup('ratings')
        .where('userId', isEqualTo: userId)
        .get();

    if (querySnapshot.docs.isEmpty) return;

    final batch = _firestore.batch();

    // 收集所有受影响的餐厅
    final affectedRestaurants = <String, List<QueryDocumentSnapshot>>{};

    for (final doc in querySnapshot.docs) {
      final restaurantId = doc.reference.parent.parent!.id;
      affectedRestaurants.putIfAbsent(restaurantId, () => []).add(doc);
    }

    // 删除所有评论
    for (final doc in querySnapshot.docs) {
      batch.delete(doc.reference);
    }

    // 更新受影响的餐厅评分
    for (final entry in affectedRestaurants.entries) {
      final restaurantId = entry.key;
      final reviews = entry.value;

      final totalRating = reviews.fold<double>(0, (sum, doc) {
        final data = doc.data() as Map<String, dynamic>;
        return sum + (data['rating']?.toDouble() ?? 0);
      });

      final averageRating = totalRating / reviews.length;

      final restaurantRef = _firestore.collection('restaurants').doc(restaurantId);
      batch.update(restaurantRef, {
        'avgRating': averageRating,
        'numRatings': FieldValue.increment(-reviews.length),
      });
    }

    await batch.commit();
  }
}
```

## 错误处理和重试

### 事务错误处理

```dart
class TransactionErrorHandler {
  static Future<T> executeWithRetry<T>(
    Future<T> Function(Transaction) transactionFunction, {
    int maxRetries = 3,
  }) async {
    int attempts = 0;

    while (attempts < maxRetries) {
      try {
        return await FirebaseFirestore.instance.runTransaction(transactionFunction);
      } on FirebaseException catch (e) {
        attempts++;

        // 检查是否可以重试
        if (_isRetryableError(e) && attempts < maxRetries) {
          final delay = Duration(seconds: attempts * 2);
          await Future.delayed(delay);
          continue;
        }

        // 重新抛出异常
        throw _getReadableError(e);
      }
    }

    throw Exception('事务执行失败，已达到最大重试次数');
  }

  static bool _isRetryableError(FirebaseException e) {
    return e.code == 'unavailable' ||
           e.code == 'deadline-exceeded' ||
           e.code == 'resource-exhausted';
  }

  static Exception _getReadableError(FirebaseException e) {
    switch (e.code) {
      case 'not-found':
        return Exception('数据不存在');
      case 'already-exists':
        return Exception('数据已存在');
      case 'permission-denied':
        return Exception('权限不足');
      case 'resource-exhausted':
        return Exception('资源耗尽，请稍后再试');
      default:
        return Exception('操作失败: ${e.message}');
    }
  }
}
```

## 总结与检查清单

通过本章的学习，你应该已经：

- [ ] 理解了子集合的概念和使用场景
- [ ] 实现了完整的评论系统
- [ ] 掌握了事务的使用方法
- [ ] 学会了批量操作的处理
- [ ] 实现了错误处理和重试机制

## 下一步

在最后一章中，我们将学习 UI 组件和导航，完成整个应用的界面设计。
