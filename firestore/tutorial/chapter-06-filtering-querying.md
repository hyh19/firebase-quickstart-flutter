# 第 6 章：查询筛选与索引

## 简介

本章将介绍 Firestore 的查询和筛选功能，以及如何创建和管理索引。我们将学习复合查询、排序、分页等高级查询技巧。

## Firestore 查询基础

### 查询类型

1. **简单查询**：单个字段条件
2. **复合查询**：多个字段条件组合
3. **范围查询**：使用比较运算符
4. **数组查询**：包含和数组成员运算

### 查询限制

- 只能在一个字段上使用不等式（`<`, `<=`, `>`, `>=`）
- 对于有不等式的查询，必须先按该字段排序
- 复合查询最多只能有 10 个子句

## 实现筛选功能

### 筛选模型

```dart
class RestaurantFilter {
  final String? category;
  final String? city;
  final int? price;
  final String? sortBy;
  final bool sortDescending;

  const RestaurantFilter({
    this.category,
    this.city,
    this.price,
    this.sortBy = 'avgRating',
    this.sortDescending = true,
  });

  bool get isActive => category != null || city != null || price != null;

  RestaurantFilter copyWith({
    String? category,
    String? city,
    int? price,
    String? sortBy,
    bool? sortDescending,
  }) {
    return RestaurantFilter(
      category: category ?? this.category,
      city: city ?? this.city,
      price: price ?? this.price,
      sortBy: sortBy ?? this.sortBy,
      sortDescending: sortDescending ?? this.sortDescending,
    );
  }
}
```

### 查询服务

```dart
class RestaurantQueryService {
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  Stream<QuerySnapshot> getFilteredRestaurants(RestaurantFilter filter) {
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

    // 应用排序
    query = query.orderBy(
      filter.sortBy!,
      descending: filter.sortDescending,
    );

    // 限制结果数量
    query = query.limit(50);

    return query.snapshots();
  }

  // 范围查询示例
  Stream<QuerySnapshot> getRestaurantsByRatingRange(double minRating, double maxRating) {
    return _firestore
        .collection('restaurants')
        .where('avgRating', isGreaterThanOrEqualTo: minRating)
        .where('avgRating', isLessThanOrEqualTo: maxRating)
        .orderBy('avgRating', descending: true)
        .snapshots();
  }

  // 数组包含查询示例
  Stream<QuerySnapshot> getRestaurantsByCategories(List<String> categories) {
    return _firestore
        .collection('restaurants')
        .where('category', whereIn: categories)
        .orderBy('avgRating', descending: true)
        .snapshots();
  }

  // 全文搜索模拟（通过多个字段查询）
  Future<List<QueryDocumentSnapshot>> searchRestaurants(String searchTerm) async {
    final term = searchTerm.toLowerCase();

    // 并行执行多个查询
    final futures = [
      _firestore
          .collection('restaurants')
          .where('name', isGreaterThanOrEqualTo: term)
          .where('name', isLessThan: term + 'z')
          .get(),
      _firestore
          .collection('restaurants')
          .where('category', isGreaterThanOrEqualTo: term)
          .where('category', isLessThan: term + 'z')
          .get(),
    ];

    final results = await Future.wait(futures);

    // 合并并去重结果
    final allDocs = <String, QueryDocumentSnapshot>{};
    for (final snapshot in results) {
      for (final doc in snapshot.docs) {
        allDocs[doc.id] = doc;
      }
    }

    return allDocs.values.toList();
  }
}
```

## 分页实现

### 游标分页

```dart
class PaginatedRestaurantService {
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  Future<PaginatedResult> getRestaurantsPage({
    required int pageSize,
    DocumentSnapshot? startAfter,
    RestaurantFilter? filter,
  }) async {
    Query query = _firestore.collection('restaurants');

    // 应用筛选条件
    if (filter != null) {
      if (filter.category != null) {
        query = query.where('category', isEqualTo: filter.category);
      }
      if (filter.city != null) {
        query = query.where('city', isEqualTo: filter.city);
      }
      if (filter.price != null) {
        query = query.where('price', isEqualTo: filter.price);
      }
    }

    // 应用排序
    query = query.orderBy('avgRating', descending: true);

    // 设置分页
    if (startAfter != null) {
      query = query.startAfterDocument(startAfter);
    }

    query = query.limit(pageSize + 1); // 多获取一个来判断是否有下一页

    final snapshot = await query.get();

    final hasNextPage = snapshot.docs.length > pageSize;
    final docs = hasNextPage
        ? snapshot.docs.sublist(0, pageSize)
        : snapshot.docs;

    return PaginatedResult(
      restaurants: docs.map((doc) => Restaurant.fromSnapshot(doc)).toList(),
      hasNextPage: hasNextPage,
      lastDocument: docs.isNotEmpty ? docs.last : null,
    );
  }
}

class PaginatedResult {
  final List<Restaurant> restaurants;
  final bool hasNextPage;
  final DocumentSnapshot? lastDocument;

  PaginatedResult({
    required this.restaurants,
    required this.hasNextPage,
    this.lastDocument,
  });
}
```

## 索引管理

### 自动索引

Firestore 会为简单查询自动创建索引，但复合查询需要手动创建索引。

### 常见索引需求

1. **复合查询索引**：多字段筛选
2. **排序索引**：按字段排序
3. **子集合索引**：子集合查询

### 索引创建

在 Firebase 控制台的 "Indexes" 标签页创建索引：

```javascript
// 示例：按类别和城市筛选，按评分排序
{
  collection: 'restaurants',
  fields: [
    { field: 'category', order: 'asc' },
    { field: 'city', order: 'asc' },
    { field: 'avgRating', order: 'desc' }
  ]
}
```

## 筛选界面

### 筛选对话框

```dart
class RestaurantFilterDialog extends StatefulWidget {
  final RestaurantFilter initialFilter;

  const RestaurantFilterDialog({
    super.key,
    required this.initialFilter,
  });

  @override
  State<RestaurantFilterDialog> createState() => _RestaurantFilterDialogState();
}

class _RestaurantFilterDialogState extends State<RestaurantFilterDialog> {
  late RestaurantFilter _filter;

  @override
  void initState() {
    super.initState();
    _filter = widget.initialFilter;
  }

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: const Text('筛选餐厅'),
      content: SingleChildScrollView(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            // 类别筛选
            DropdownButtonFormField<String>(
              value: _filter.category,
              decoration: const InputDecoration(labelText: '类别'),
              items: ['中餐', '西餐', '日料', '快餐', '咖啡厅']
                  .map((category) => DropdownMenuItem(
                        value: category,
                        child: Text(category),
                      ))
                  .toList(),
              onChanged: (value) {
                setState(() => _filter = _filter.copyWith(category: value));
              },
            ),

            // 城市筛选
            DropdownButtonFormField<String>(
              value: _filter.city,
              decoration: const InputDecoration(labelText: '城市'),
              items: ['北京', '上海', '深圳', '广州']
                  .map((city) => DropdownMenuItem(
                        value: city,
                        child: Text(city),
                      ))
                  .toList(),
              onChanged: (value) {
                setState(() => _filter = _filter.copyWith(city: value));
              },
            ),

            // 价格筛选
            DropdownButtonFormField<int>(
              value: _filter.price,
              decoration: const InputDecoration(labelText: '价格等级'),
              items: [
                const DropdownMenuItem(value: 1, child: Text('经济实惠')),
                const DropdownMenuItem(value: 2, child: Text('中等价格')),
                const DropdownMenuItem(value: 3, child: Text('高档消费')),
              ],
              onChanged: (value) {
                setState(() => _filter = _filter.copyWith(price: value));
              },
            ),

            // 排序选项
            DropdownButtonFormField<String>(
              value: _filter.sortBy,
              decoration: const InputDecoration(labelText: '排序方式'),
              items: [
                const DropdownMenuItem(value: 'avgRating', child: Text('按评分')),
                const DropdownMenuItem(value: 'name', child: Text('按名称')),
                const DropdownMenuItem(value: 'price', child: Text('按价格')),
              ],
              onChanged: (value) {
                setState(() => _filter = _filter.copyWith(sortBy: value));
              },
            ),
          ],
        ),
      ),
      actions: [
        TextButton(
          onPressed: () {
            Navigator.of(context).pop(RestaurantFilter()); // 重置筛选
          },
          child: const Text('重置'),
        ),
        TextButton(
          onPressed: () => Navigator.of(context).pop(_filter),
          child: const Text('应用'),
        ),
      ],
    );
  }
}
```

## 查询性能优化

### 查询优化技巧

1. **使用复合索引**：为常用查询组合创建索引
2. **限制结果数量**：使用 `limit()` 避免返回过多数据
3. **选择性查询**：优先使用高选择性的字段进行筛选
4. **避免全集合扫描**：始终使用 where 子句限制查询范围

### 性能监控

```dart
class QueryPerformanceMonitor {
  final Map<String, QueryMetrics> _metrics = {};

  void trackQuery(String queryName, Query query) {
    final startTime = DateTime.now();

    query.get().then((snapshot) {
      final duration = DateTime.now().difference(startTime);
      _metrics[queryName] = QueryMetrics(
        duration: duration,
        documentCount: snapshot.docs.length,
        timestamp: DateTime.now(),
      );
    });
  }

  void printMetrics() {
    _metrics.forEach((name, metrics) {
      print('$name: ${metrics.duration.inMilliseconds}ms, ${metrics.documentCount} docs');
    });
  }
}

class QueryMetrics {
  final Duration duration;
  final int documentCount;
  final DateTime timestamp;

  QueryMetrics({
    required this.duration,
    required this.documentCount,
    required this.timestamp,
  });
}
```

## 总结与检查清单

通过本章的学习，你应该已经：

- [ ] 理解了 Firestore 查询的基本概念和限制
- [ ] 实现了复合查询和范围查询
- [ ] 创建了分页查询功能
- [ ] 学会了创建和管理索引
- [ ] 实现了用户友好的筛选界面
- [ ] 掌握了查询性能优化技巧

## 下一步

在下一章中，我们将学习评论子集合和事务操作，了解如何处理复杂的数据关系。
