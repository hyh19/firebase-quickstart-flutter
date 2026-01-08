# 第 5 章：实时数据流与监听器

## 简介

本章将介绍如何使用 Firestore 的实时数据流功能，让应用能够自动响应数据的变化。我们将学习如何设置监听器、管理数据流，以及处理实时更新的用户界面。

## StreamBuilder 基础

### 什么是 StreamBuilder

`StreamBuilder` 是 Flutter 中用于处理异步数据流的组件，它会自动重建 UI 以响应数据变化。

```dart
StreamBuilder<T>(
  stream: myStream,
  initialData: initialValue,
  builder: (BuildContext context, AsyncSnapshot<T> snapshot) {
    if (snapshot.hasError) {
      return Text('错误: ${snapshot.error}');
    }

    if (!snapshot.hasData) {
      return const CircularProgressIndicator();
    }

    return MyWidget(data: snapshot.data!);
  },
)
```

## 实现实时数据监听器

### 创建实时数据服务

```dart
import 'dart:async';
import 'package:cloud_firestore/cloud_firestore.dart';

class RealtimeRestaurantService {
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  /// 获取餐厅列表流
  Stream<List<Map<String, dynamic>>> getRestaurantsStream() {
    return _firestore
        .collection('restaurants')
        .orderBy('avgRating', descending: true)
        .limit(50)
        .snapshots()
        .map((snapshot) =>
            snapshot.docs.map((doc) => doc.data()).toList());
  }

  /// 获取单个餐厅流
  Stream<Map<String, dynamic>?> getRestaurantStream(String restaurantId) {
    return _firestore
        .collection('restaurants')
        .doc(restaurantId)
        .snapshots()
        .map((doc) => doc.data());
  }

  /// 获取餐厅评论流
  Stream<List<Map<String, dynamic>>> getReviewsStream(String restaurantId) {
    return _firestore
        .collection('restaurants')
        .doc(restaurantId)
        .collection('ratings')
        .orderBy('timestamp', descending: true)
        .snapshots()
        .map((snapshot) =>
            snapshot.docs.map((doc) => doc.data()).toList());
  }

  /// 监听特定查询条件的变化
  Stream<QuerySnapshot> getFilteredRestaurantsStream({
    String? category,
    String? city,
    int? price,
  }) {
    Query query = _firestore.collection('restaurants');

    if (category != null) {
      query = query.where('category', isEqualTo: category);
    }
    if (city != null) {
      query = query.where('city', isEqualTo: city);
    }
    if (price != null) {
      query = query.where('price', isEqualTo: price);
    }

    return query
        .orderBy('avgRating', descending: true)
        .limit(50)
        .snapshots();
  }
}
```

## 创建实时 UI 组件

### 餐厅列表组件

```dart
import 'package:flutter/material.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

class RealtimeRestaurantList extends StatelessWidget {
  final Stream<QuerySnapshot> restaurantsStream;

  const RealtimeRestaurantList({
    super.key,
    required this.restaurantsStream,
  });

  @override
  Widget build(BuildContext context) {
    return StreamBuilder<QuerySnapshot>(
      stream: restaurantsStream,
      builder: (context, snapshot) {
        if (snapshot.hasError) {
          return Center(
            child: Text('加载失败: ${snapshot.error}'),
          );
        }

        if (snapshot.connectionState == ConnectionState.waiting) {
          return const Center(child: CircularProgressIndicator());
        }

        final docs = snapshot.data?.docs ?? [];

        if (docs.isEmpty) {
          return const Center(
            child: Text('暂无餐厅数据'),
          );
        }

        return ListView.builder(
          itemCount: docs.length,
          itemBuilder: (context, index) {
            final data = docs[index].data() as Map<String, dynamic>;
            return RestaurantListItem(
              restaurant: Restaurant.fromMap(data),
              onTap: () => _navigateToDetail(context, docs[index].id),
            );
          },
        );
      },
    );
  }

  void _navigateToDetail(BuildContext context, String restaurantId) {
    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => RestaurantDetailPage(restaurantId: restaurantId),
      ),
    );
  }
}
```

### 实时评论列表

```dart
class RealtimeReviewList extends StatelessWidget {
  final String restaurantId;

  const RealtimeReviewList({
    super.key,
    required this.restaurantId,
  });

  @override
  Widget build(BuildContext context) {
    final reviewsStream = FirebaseFirestore.instance
        .collection('restaurants')
        .doc(restaurantId)
        .collection('ratings')
        .orderBy('timestamp', descending: true)
        .snapshots();

    return StreamBuilder<QuerySnapshot>(
      stream: reviewsStream,
      builder: (context, snapshot) {
        if (snapshot.hasError) {
          return Text('加载评论失败: ${snapshot.error}');
        }

        if (snapshot.connectionState == ConnectionState.waiting) {
          return const CircularProgressIndicator();
        }

        final reviews = snapshot.data?.docs ?? [];

        if (reviews.isEmpty) {
          return const Text('暂无评论');
        }

        return ListView.builder(
          shrinkWrap: true,
          physics: const NeverScrollableScrollPhysics(),
          itemCount: reviews.length,
          itemBuilder: (context, index) {
            final data = reviews[index].data() as Map<String, dynamic>;
            return ReviewItem(review: Review.fromMap(data));
          },
        );
      },
    );
  }
}
```

## 连接状态管理

### 处理网络连接状态

```dart
import 'package:connectivity_plus/connectivity_plus.dart';

class ConnectivityService {
  final Connectivity _connectivity = Connectivity();

  Stream<ConnectivityResult> get connectivityStream =>
      _connectivity.onConnectivityChanged;

  Future<ConnectivityResult> get currentConnectivity =>
      _connectivity.checkConnectivity();
}
```

### 带连接状态的实时组件

```dart
class ConnectionAwareRestaurantList extends StatefulWidget {
  const ConnectionAwareRestaurantList({super.key});

  @override
  State<ConnectionAwareRestaurantList> createState() =>
      _ConnectionAwareRestaurantListState();
}

class _ConnectionAwareRestaurantListState
    extends State<ConnectionAwareRestaurantList> {
  late final ConnectivityService _connectivityService;
  late final StreamSubscription<ConnectivityResult> _connectivitySubscription;

  ConnectivityResult _connectionStatus = ConnectivityResult.none;
  bool _isOnline = false;

  @override
  void initState() {
    super.initState();
    _connectivityService = ConnectivityService();
    _initConnectivity();

    _connectivitySubscription =
        _connectivityService.connectivityStream.listen(_updateConnectionStatus);
  }

  @override
  void dispose() {
    _connectivitySubscription.cancel();
    super.dispose();
  }

  Future<void> _initConnectivity() async {
    final result = await _connectivityService.currentConnectivity;
    _updateConnectionStatus(result);
  }

  void _updateConnectionStatus(ConnectivityResult result) {
    setState(() {
      _connectionStatus = result;
      _isOnline = result != ConnectivityResult.none;
    });
  }

  @override
  Widget build(BuildContext context) {
    if (!_isOnline) {
      return const Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(Icons.wifi_off, size: 64, color: Colors.grey),
            SizedBox(height: 16),
            Text('无网络连接'),
            Text('请检查网络设置'),
          ],
        ),
      );
    }

    return Stack(
      children: [
        RealtimeRestaurantList(
          restaurantsStream: FirebaseFirestore.instance
              .collection('restaurants')
              .orderBy('avgRating', descending: true)
              .snapshots(),
        ),
        if (_connectionStatus == ConnectivityResult.mobile)
          Positioned(
            top: 0,
            left: 0,
            right: 0,
            child: Container(
              color: Colors.orange,
              padding: const EdgeInsets.all(4),
              child: const Text(
                '正在使用移动数据',
                textAlign: TextAlign.center,
                style: TextStyle(color: Colors.white),
              ),
            ),
          ),
      ],
    );
  }
}
```

## 性能优化

### 限制监听器数量

```dart
class StreamManager {
  final Map<String, StreamSubscription> _subscriptions = {};

  void addSubscription(String key, StreamSubscription subscription) {
    cancelSubscription(key);
    _subscriptions[key] = subscription;
  }

  void cancelSubscription(String key) {
    _subscriptions[key]?.cancel();
    _subscriptions.remove(key);
  }

  void cancelAll() {
    for (final subscription in _subscriptions.values) {
      subscription.cancel();
    }
    _subscriptions.clear();
  }
}
```

### 数据缓存策略

```dart
class CachedRestaurantProvider {
  final Map<String, Restaurant> _cache = {};
  final Duration _cacheDuration = const Duration(minutes: 5);

  Restaurant? getCachedRestaurant(String id) {
    final cached = _cache[id];
    if (cached != null) {
      // 检查缓存是否过期
      final now = DateTime.now();
      final cacheTime = cached.cacheTime ?? now.subtract(_cacheDuration);
      if (now.difference(cacheTime) < _cacheDuration) {
        return cached;
      } else {
        _cache.remove(id);
      }
    }
    return null;
  }

  void cacheRestaurant(Restaurant restaurant) {
    _cache[restaurant.id!] = restaurant.copyWith(cacheTime: DateTime.now());
  }
}
```

## 总结与检查清单

通过本章的学习，你应该已经：

- [ ] 理解了 StreamBuilder 的工作原理
- [ ] 实现了实时数据监听器
- [ ] 创建了响应式用户界面组件
- [ ] 处理了网络连接状态
- [ ] 实现了基本的性能优化

## 下一步

在下一章中，我们将学习查询筛选和索引，为应用添加高级搜索功能。
