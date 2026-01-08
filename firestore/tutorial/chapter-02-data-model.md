# 第 2 章：数据模型与实体类

## 简介

本章将介绍如何设计和实现 Firestore 数据模型。我们将学习如何创建 Dart 类来表示 Firestore 文档，以及如何在 Dart 对象和 Firestore 数据之间进行转换。

## Firestore 数据结构设计

在 FriendlyEats 应用中，我们需要存储两种主要数据类型：

1. **餐厅 (Restaurant)** - 主文档
2. **评论 (Review)** - 餐厅的子集合文档

### 餐厅数据结构

```json
{
  "name": "Italian Kitchen",
  "category": "Italian",
  "city": "San Francisco",
  "avgRating": 4.2,
  "numRatings": 15,
  "price": 2,
  "photo": "https://storage.googleapis.com/.../food_1.png"
}
```

### 评论数据结构

```json
{
  "userId": "user123",
  "userName": "John Doe",
  "rating": 5.0,
  "text": "Amazing food and great service!",
  "timestamp": "2024-01-01T10:00:00Z"
}
```

## 实现 Restaurant 模型

创建 `lib/models/restaurant.dart` 文件：

```dart
import 'dart:math';

import 'package:cloud_firestore/cloud_firestore.dart';

/// 餐厅数据模型
class Restaurant {
  /// 文档 ID（由 Firestore 自动生成）
  final String? id;

  /// 餐厅名称
  final String name;

  /// 餐厅类别（如：Italian, Chinese, Mexican 等）
  final String? category;

  /// 所在城市
  final String? city;

  /// 平均评分
  final double avgRating;

  /// 评分数量
  final int numRatings;

  /// 价格等级（1-3，1 最便宜）
  final int? price;

  /// 餐厅照片 URL
  final String? photo;

  /// Firestore 文档引用
  final DocumentReference? reference;

  /// 私有构造函数
  Restaurant._({
    required this.name,
    this.category,
    this.city,
    this.price,
    this.photo,
    this.id,
    this.numRatings = 0,
    this.avgRating = 0,
    this.reference,
  });

  /// 从 Firestore DocumentSnapshot 创建 Restaurant 对象
  factory Restaurant.fromSnapshot(DocumentSnapshot snapshot) {
    final data = snapshot.data() as Map<String, dynamic>;
    return Restaurant._(
      id: snapshot.id,
      name: data['name'] ?? '',
      category: data['category'],
      city: data['city'],
      avgRating: (data['avgRating'] ?? 0).toDouble(),
      numRatings: data['numRatings'] ?? 0,
      price: data['price'],
      photo: data['photo'],
      reference: snapshot.reference,
    );
  }

  /// 创建随机餐厅（用于测试数据）
  factory Restaurant.random() {
    return Restaurant._(
      category: _getRandomCategory(),
      city: _getRandomCity(),
      name: _getRandomName(),
      price: Random().nextInt(3) + 1, // 1-3 之间的随机数
      photo: _getRandomPhoto(),
    );
  }

  /// 将 Restaurant 对象转换为 Map（用于保存到 Firestore）
  Map<String, dynamic> toMap() {
    return {
      'name': name,
      'category': category,
      'city': city,
      'avgRating': avgRating,
      'numRatings': numRatings,
      'price': price,
      'photo': photo,
    };
  }

  /// 复制并修改某些字段
  Restaurant copyWith({
    String? id,
    String? name,
    String? category,
    String? city,
    double? avgRating,
    int? numRatings,
    int? price,
    String? photo,
    DocumentReference? reference,
  }) {
    return Restaurant._(
      id: id ?? this.id,
      name: name ?? this.name,
      category: category ?? this.category,
      city: city ?? this.city,
      avgRating: avgRating ?? this.avgRating,
      numRatings: numRatings ?? this.numRatings,
      price: price ?? this.price,
      photo: photo ?? this.photo,
      reference: reference ?? this.reference,
    );
  }

  @override
  String toString() {
    return 'Restaurant(id: $id, name: $name, category: $category, city: $city, '
           'avgRating: $avgRating, numRatings: $numRatings, price: $price)';
  }

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is Restaurant && other.id == id;
  }

  @override
  int get hashCode => id.hashCode;
}

// 辅助函数
String _getRandomName() {
  final words = ['Pizza', 'Burger', 'Pasta', 'Sushi', 'Taco', 'Salad', 'Steak'];
  final adjectives = ['Delicious', 'Tasty', 'Fresh', 'Hot', 'Spicy', 'Sweet'];

  final adjective = adjectives[Random().nextInt(adjectives.length)];
  final word = words[Random().nextInt(words.length)];

  return '$adjective $word';
}

String _getRandomCategory() {
  final categories = ['Italian', 'American', 'Chinese', 'Mexican', 'Japanese', 'Thai'];
  return categories[Random().nextInt(categories.length)];
}

String _getRandomCity() {
  final cities = ['New York', 'Los Angeles', 'Chicago', 'Houston', 'Phoenix', 'San Francisco'];
  return cities[Random().nextInt(cities.length)];
}

String _getRandomPhoto() {
  final photoId = Random().nextInt(20) + 1;
  return 'https://storage.googleapis.com/firestorequickstarts.appspot.com/food_$photoId.png';
}
```

## 实现 Review 模型

创建 `lib/models/review.dart` 文件：

```dart
import 'dart:math';

import 'package:cloud_firestore/cloud_firestore.dart';

/// 评论/评分数据模型
class Review {
  /// 文档 ID（由 Firestore 自动生成）
  late final String id;

  /// 用户 ID
  final String userId;

  /// 用户名
  final String userName;

  /// 评分（1.0 - 5.0）
  final double rating;

  /// 评论文本
  final String text;

  /// 评论时间戳
  late final Timestamp timestamp;

  /// Firestore 文档引用
  late final DocumentReference reference;

  /// 私有构造函数
  Review._({
    required this.id,
    required this.userId,
    required this.userName,
    required this.rating,
    required this.text,
    required this.timestamp,
    required this.reference,
  });

  /// 从 Firestore DocumentSnapshot 创建 Review 对象
  factory Review.fromSnapshot(DocumentSnapshot snapshot) {
    final data = snapshot.data() as Map<String, dynamic>;
    return Review._(
      id: snapshot.id,
      userId: data['userId'] ?? '',
      userName: data['userName'] ?? 'Anonymous',
      rating: (data['rating'] ?? 0).toDouble(),
      text: data['text'] ?? '',
      timestamp: data['timestamp'] ?? Timestamp.now(),
      reference: snapshot.reference,
    );
  }

  /// 从用户输入创建 Review 对象
  Review.fromUserInput({
    required this.rating,
    required this.text,
    required this.userName,
    required this.userId,
  }) : timestamp = Timestamp.now();

  /// 创建随机评论（用于测试）
  factory Review.random({
    required String userName,
    required String userId,
  }) {
    final rating = Random().nextInt(5) + 1; // 1-5 之间的随机数
    final text = _getRandomReviewText(rating);
    return Review.fromUserInput(
      rating: rating.toDouble(),
      text: text,
      userName: userName,
      userId: userId,
    );
  }

  /// 将 Review 对象转换为 Map（用于保存到 Firestore）
  Map<String, dynamic> toMap() {
    return {
      'userId': userId,
      'userName': userName,
      'rating': rating,
      'text': text,
      'timestamp': timestamp,
    };
  }

  /// 判断是否为正面评价
  bool get isPositive => rating >= 3.0;

  /// 获取评分的星星图标
  String get stars => '★' * rating.round() + '☆' * (5 - rating.round());

  @override
  String toString() {
    return 'Review(id: $id, userName: $userName, rating: $rating, text: $text)';
  }

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is Review && other.id == id;
  }

  @override
  int get hashCode => id.hashCode;
}

// 辅助函数
String _getRandomReviewText(int rating) {
  final reviews = {
    1: [
      '非常糟糕的体验，不会再来了！',
      '食物难吃，服务态度差。',
      '完全不符合预期。'
    ],
    2: [
      '一般般吧，不是很满意。',
      '食物还可以，但服务需要改进。',
      '平淡无奇的体验。'
    ],
    3: [
      '还可以，符合预期。',
      '食物一般，价格合理。',
      '中规中矩的一家餐厅。'
    ],
    4: [
      '很不错的餐厅，值得推荐！',
      '食物美味，服务周到。',
      '经常来的地方，每次都很满意。'
    ],
    5: [
      '完美的餐厅体验！',
      '这是我最喜欢的餐厅之一。',
      '强烈推荐给所有朋友！'
    ],
  };

  final reviewTexts = reviews[rating] ?? ['一般评价'];
  return reviewTexts[Random().nextInt(reviewTexts.length)];
}
```

## 实现 Filter 模型

创建 `lib/models/filter.dart` 文件：

```dart
/// 餐厅筛选条件
class Filter {
  /// 城市筛选
  final String? city;

  /// 价格等级筛选（1-3）
  final int? price;

  /// 类别筛选
  final String? category;

  /// 排序字段
  final String? sort;

  /// 构造函数
  const Filter({
    this.city,
    this.price,
    this.category,
    this.sort,
  });

  /// 判断是否为默认筛选（无筛选条件）
  bool get isDefault {
    return city == null && price == null && category == null && sort == null;
  }

  /// 判断是否有活跃的筛选条件
  bool get hasActiveFilters {
    return !isDefault;
  }

  /// 获取活跃筛选条件的数量
  int get activeFilterCount {
    int count = 0;
    if (city != null) count++;
    if (price != null) count++;
    if (category != null) count++;
    return count;
  }

  /// 创建新的 Filter 对象，修改指定字段
  Filter copyWith({
    String? city,
    int? price,
    String? category,
    String? sort,
  }) {
    return Filter(
      city: city ?? this.city,
      price: price ?? this.price,
      category: category ?? this.category,
      sort: sort ?? this.sort,
    );
  }

  /// 重置所有筛选条件
  Filter reset() {
    return const Filter();
  }

  @override
  String toString() {
    return 'Filter(city: $city, price: $price, category: $category, sort: $sort)';
  }

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is Filter &&
           other.city == city &&
           other.price == price &&
           other.category == category &&
           other.sort == sort;
  }

  @override
  int get hashCode {
    return Object.hash(city, price, category, sort);
  }
}

/// 筛选条件变化的回调类型
typedef FilterChangedCallback = void Function(Filter filter);
```

## 创建常量和工具类

创建 `lib/models/constants.dart` 文件来存储常量数据：

```dart
/// 应用中使用的常量数据
class AppConstants {
  /// 支持的城市列表
  static const List<String> cities = [
    '北京', '上海', '深圳', '广州', '杭州', '南京', '苏州', '成都', '武汉', '西安',
    '纽约', '洛杉矶', '芝加哥', '休斯顿', '菲尼克斯', '费城', '圣安东尼奥', '圣迭戈',
  ];

  /// 餐厅类别列表
  static const List<String> categories = [
    '中餐', '西餐', '日料', '韩餐', '泰餐', '意大利菜', '墨西哥菜', '美式快餐',
    '咖啡厅', '甜品店', '火锅', '烧烤', '海鲜', '素食', '快餐',
  ];

  /// 排序选项
  static const List<String> sortOptions = [
    'avgRating', // 按评分排序
    'name',      // 按名称排序
    'price',     // 按价格排序
  ];

  /// 价格等级标签
  static const Map<int, String> priceLabels = {
    1: '经济实惠',
    2: '中等价格',
    3: '高档消费',
  };

  /// 评分标签
  static const Map<double, String> ratingLabels = {
    1.0: '非常差',
    2.0: '较差',
    3.0: '一般',
    4.0: '良好',
    5.0: '优秀',
  };
}
```

## 数据验证

添加数据验证逻辑到模型类中：

```dart
/// 数据验证异常
class ValidationException implements Exception {
  final String message;
  ValidationException(this.message);

  @override
  String toString() => 'ValidationException: $message';
}

/// Restaurant 数据验证
extension RestaurantValidation on Restaurant {
  /// 验证餐厅数据
  void validate() {
    if (name.trim().isEmpty) {
      throw ValidationException('餐厅名称不能为空');
    }
    if (name.length > 100) {
      throw ValidationException('餐厅名称不能超过100个字符');
    }
    if (avgRating < 0 || avgRating > 5) {
      throw ValidationException('平均评分必须在0-5之间');
    }
    if (numRatings < 0) {
      throw ValidationException('评分数量不能为负数');
    }
    if (price != null && (price! < 1 || price! > 3)) {
      throw ValidationException('价格等级必须在1-3之间');
    }
  }
}

/// Review 数据验证
extension ReviewValidation on Review {
  /// 验证评论数据
  void validate() {
    if (userId.trim().isEmpty) {
      throw ValidationException('用户ID不能为空');
    }
    if (userName.trim().isEmpty) {
      throw ValidationException('用户名不能为空');
    }
    if (rating < 1 || rating > 5) {
      throw ValidationException('评分必须在1-5之间');
    }
    if (text.trim().isEmpty) {
      throw ValidationException('评论内容不能为空');
    }
    if (text.length > 1000) {
      throw ValidationException('评论内容不能超过1000个字符');
    }
  }
}
```

## 实践练习

1. **创建模型类**：在你的项目中创建上述三个模型类文件
2. **添加验证**：为每个模型类添加数据验证扩展
3. **测试数据创建**：创建一些测试数据来验证模型是否正常工作

```dart
// 测试代码
void main() {
  // 创建餐厅
  final restaurant = Restaurant.random();
  print('随机餐厅: $restaurant');

  // 创建评论
  final review = Review.fromUserInput(
    rating: 4.5,
    text: '很棒的餐厅！',
    userName: '张三',
    userId: 'user123',
  );
  print('用户评论: $review');

  // 创建筛选条件
  final filter = Filter(
    city: '北京',
    category: '中餐',
    sort: 'avgRating',
  );
  print('筛选条件: $filter');
}
```

## 总结与检查清单

通过本章的学习，你应该已经：

- [ ] 理解了 Firestore 数据结构设计原则
- [ ] 实现了 Restaurant、Review 和 Filter 模型类
- [ ] 掌握了从 Firestore DocumentSnapshot 创建对象的方法
- [ ] 学会了将对象转换为 Map 的方法
- [ ] 添加了数据验证逻辑
- [ ] 创建了常量和工具类

## 下一步

在下一章中，我们将学习如何实现 Firestore 的基本 CRUD 操作，包括创建、读取、更新和删除文档。
