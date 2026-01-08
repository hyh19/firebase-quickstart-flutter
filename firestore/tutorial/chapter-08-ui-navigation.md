# 第 8 章：UI 组件与导航

## 简介

本章将介绍如何构建完整的 Flutter UI，包括导航、列表显示、表单输入等。我们将创建一个完整的餐厅推荐应用界面。

## 应用架构

### 主要页面

1. **首页**：餐厅列表和筛选
2. **餐厅详情页**：餐厅信息和评论
3. **添加餐厅页**：创建新餐厅
4. **添加评论页**：为餐厅添加评论

### 导航结构

```text
HomePage (首页)
├── RestaurantList (餐厅列表)
├── FilterBar (筛选栏)
└── FloatingActionButton (添加按钮)

RestaurantDetailPage (餐厅详情)
├── RestaurantInfo (餐厅信息)
├── ReviewList (评论列表)
└── AddReviewButton (添加评论按钮)
```

## 实现导航

### 路由配置

```dart
class AppRoutes {
  static const home = '/';
  static const restaurantDetail = '/restaurant';
  static const addRestaurant = '/add-restaurant';
  static const addReview = '/add-review';
}

class AppRouter {
  static Route<dynamic> generateRoute(RouteSettings settings) {
    switch (settings.name) {
      case AppRoutes.home:
        return MaterialPageRoute(builder: (_) => const HomePage());

      case AppRoutes.restaurantDetail:
        final args = settings.arguments as RestaurantDetailArguments;
        return MaterialPageRoute(
          builder: (_) => RestaurantDetailPage(restaurant: args.restaurant),
        );

      case AppRoutes.addRestaurant:
        return MaterialPageRoute(builder: (_) => const AddRestaurantPage());

      case AppRoutes.addReview:
        final args = settings.arguments as AddReviewArguments;
        return MaterialPageRoute(
          builder: (_) => AddReviewPage(restaurantId: args.restaurantId),
        );

      default:
        return MaterialPageRoute(
          builder: (_) => const Scaffold(
            body: Center(child: Text('页面不存在')),
          ),
        );
    }
  }
}

class RestaurantDetailArguments {
  final Restaurant restaurant;
  RestaurantDetailArguments(this.restaurant);
}

class AddReviewArguments {
  final String restaurantId;
  AddReviewArguments(this.restaurantId);
}
```

## 首页实现

### 餐厅列表项组件

```dart
class RestaurantListItem extends StatelessWidget {
  final Restaurant restaurant;
  final VoidCallback onTap;

  const RestaurantListItem({
    super.key,
    required this.restaurant,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      child: InkWell(
        onTap: onTap,
        borderRadius: BorderRadius.circular(8),
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Row(
            children: [
              // 餐厅图片
              Container(
                width: 80,
                height: 80,
                decoration: BoxDecoration(
                  borderRadius: BorderRadius.circular(8),
                  image: DecorationImage(
                    image: NetworkImage(restaurant.photo ??
                        'https://via.placeholder.com/80x80?text=No+Image'),
                    fit: BoxFit.cover,
                  ),
                ),
              ),

              const SizedBox(width: 16),

              // 餐厅信息
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      restaurant.name,
                      style: Theme.of(context).textTheme.titleMedium?.copyWith(
                        fontWeight: FontWeight.bold,
                      ),
                    ),

                    const SizedBox(height: 4),

                    Text(
                      restaurant.category ?? '未分类',
                      style: Theme.of(context).textTheme.bodyMedium?.copyWith(
                        color: Colors.grey[600],
                      ),
                    ),

                    const SizedBox(height: 4),

                    Row(
                      children: [
                        Icon(Icons.location_on, size: 16, color: Colors.grey),
                        const SizedBox(width: 4),
                        Text(
                          restaurant.city ?? '未知城市',
                          style: Theme.of(context).textTheme.bodySmall,
                        ),
                      ],
                    ),

                    const SizedBox(height: 8),

                    // 评分和价格
                    Row(
                      children: [
                        // 评分
                        Row(
                          children: [
                            Icon(Icons.star, size: 16, color: Colors.amber),
                            const SizedBox(width: 4),
                            Text(
                              restaurant.avgRating.toStringAsFixed(1),
                              style: Theme.of(context).textTheme.bodyMedium,
                            ),
                            Text(
                              ' (${restaurant.numRatings})',
                              style: Theme.of(context).textTheme.bodySmall?.copyWith(
                                color: Colors.grey,
                              ),
                            ),
                          ],
                        ),

                        const Spacer(),

                        // 价格等级
                        if (restaurant.price != null)
                          Row(
                            children: List.generate(
                              3,
                              (index) => Icon(
                                Icons.attach_money,
                                size: 16,
                                color: index < restaurant.price!
                                    ? Colors.green
                                    : Colors.grey[300],
                              ),
                            ),
                          ),
                      ],
                    ),
                  ],
                ),
              ),

              // 右箭头
              Icon(Icons.chevron_right, color: Colors.grey),
            ],
          ),
        ),
      ),
    );
  }
}
```

### 筛选栏组件

```dart
class FilterBar extends StatelessWidget {
  final RestaurantFilter filter;
  final VoidCallback onFilterPressed;
  final VoidCallback onSortPressed;

  const FilterBar({
    super.key,
    required this.filter,
    required this.onFilterPressed,
    required this.onSortPressed,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      color: Colors.white,
      child: Row(
        children: [
          // 筛选按钮
          Expanded(
            child: OutlinedButton.icon(
              onPressed: onFilterPressed,
              icon: const Icon(Icons.filter_list),
              label: Text(_getFilterText()),
              style: OutlinedButton.styleFrom(
                alignment: Alignment.centerLeft,
              ),
            ),
          ),

          const SizedBox(width: 8),

          // 排序按钮
          IconButton(
            onPressed: onSortPressed,
            icon: const Icon(Icons.sort),
            tooltip: '排序',
          ),
        ],
      ),
    );
  }

  String _getFilterText() {
    if (!filter.isActive) {
      return '筛选';
    }

    final parts = <String>[];
    if (filter.category != null) parts.add(filter.category!);
    if (filter.city != null) parts.add(filter.city!);
    if (filter.price != null) parts.add('\$${filter.price}');

    return parts.join(', ');
  }
}
```

## 餐厅详情页面

### 餐厅信息组件

```dart
class RestaurantDetailHeader extends StatelessWidget {
  final Restaurant restaurant;

  const RestaurantDetailHeader({
    super.key,
    required this.restaurant,
  });

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        // 餐厅图片
        Container(
          height: 200,
          width: double.infinity,
          decoration: BoxDecoration(
            image: DecorationImage(
              image: NetworkImage(restaurant.photo ??
                  'https://via.placeholder.com/400x200?text=No+Image'),
              fit: BoxFit.cover,
            ),
          ),
        ),

        Padding(
          padding: const EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              // 餐厅名称
              Text(
                restaurant.name,
                style: Theme.of(context).textTheme.headlineSmall?.copyWith(
                  fontWeight: FontWeight.bold,
                ),
              ),

              const SizedBox(height: 8),

              // 类别和城市
              Row(
                children: [
                  Container(
                    padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
                    decoration: BoxDecoration(
                      color: Colors.blue[100],
                      borderRadius: BorderRadius.circular(16),
                    ),
                    child: Text(
                      restaurant.category ?? '未分类',
                      style: TextStyle(color: Colors.blue[800]),
                    ),
                  ),

                  const SizedBox(width: 8),

                  Icon(Icons.location_on, size: 16, color: Colors.grey),
                  const SizedBox(width: 4),
                  Text(
                    restaurant.city ?? '未知城市',
                    style: Theme.of(context).textTheme.bodyMedium,
                  ),
                ],
              ),

              const SizedBox(height: 16),

              // 评分信息
              Row(
                children: [
                  Icon(Icons.star, color: Colors.amber, size: 24),
                  const SizedBox(width: 8),
                  Text(
                    restaurant.avgRating.toStringAsFixed(1),
                    style: Theme.of(context).textTheme.headlineSmall,
                  ),
                  Text(
                    ' (${restaurant.numRatings} 条评论)',
                    style: Theme.of(context).textTheme.bodyMedium?.copyWith(
                      color: Colors.grey,
                    ),
                  ),
                ],
              ),

              const SizedBox(height: 8),

              // 价格等级
              if (restaurant.price != null)
                Row(
                  children: [
                    Text('价格等级: ', style: Theme.of(context).textTheme.bodyMedium),
                    ...List.generate(
                      3,
                      (index) => Icon(
                        Icons.attach_money,
                        size: 16,
                        color: index < restaurant.price! ? Colors.green : Colors.grey[300],
                      ),
                    ),
                  ],
                ),
            ],
          ),
        ),
      ],
    );
  }
}
```

### 评论列表组件

```dart
class ReviewList extends StatelessWidget {
  final Stream<List<Review>> reviewsStream;

  const ReviewList({
    super.key,
    required this.reviewsStream,
  });

  @override
  Widget build(BuildContext context) {
    return StreamBuilder<List<Review>>(
      stream: reviewsStream,
      builder: (context, snapshot) {
        if (snapshot.hasError) {
          return Center(
            child: Text('加载评论失败: ${snapshot.error}'),
          );
        }

        if (!snapshot.hasData) {
          return const Center(child: CircularProgressIndicator());
        }

        final reviews = snapshot.data!;

        if (reviews.isEmpty) {
          return const Center(
            child: Text('暂无评论'),
          );
        }

        return ListView.builder(
          shrinkWrap: true,
          physics: const NeverScrollableScrollPhysics(),
          itemCount: reviews.length,
          itemBuilder: (context, index) {
            return ReviewItem(review: reviews[index]);
          },
        );
      },
    );
  }
}

class ReviewItem extends StatelessWidget {
  final Review review;

  const ReviewItem({
    super.key,
    required this.review,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // 用户名和评分
            Row(
              children: [
                CircleAvatar(
                  backgroundColor: Colors.blue[100],
                  child: Text(
                    review.userName.isNotEmpty ? review.userName[0].toUpperCase() : '?',
                    style: TextStyle(color: Colors.blue[800]),
                  ),
                ),

                const SizedBox(width: 12),

                Expanded(
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text(
                        review.userName,
                        style: Theme.of(context).textTheme.titleSmall?.copyWith(
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                      Row(
                        children: [
                          ...List.generate(
                            5,
                            (index) => Icon(
                              index < review.rating.round()
                                  ? Icons.star
                                  : Icons.star_border,
                              size: 16,
                              color: Colors.amber,
                            ),
                          ),
                          const SizedBox(width: 8),
                          Text(
                            review.rating.toStringAsFixed(1),
                            style: Theme.of(context).textTheme.bodySmall,
                          ),
                        ],
                      ),
                    ],
                  ),
                ),

                Text(
                  _formatDate(review.timestamp),
                  style: Theme.of(context).textTheme.bodySmall?.copyWith(
                    color: Colors.grey,
                  ),
                ),
              ],
            ),

            const SizedBox(height: 12),

            // 评论内容
            Text(
              review.text,
              style: Theme.of(context).textTheme.bodyMedium,
            ),
          ],
        ),
      ),
    );
  }

  String _formatDate(DateTime date) {
    final now = DateTime.now();
    final difference = now.difference(date);

    if (difference.inDays > 0) {
      return '${difference.inDays}天前';
    } else if (difference.inHours > 0) {
      return '${difference.inHours}小时前';
    } else if (difference.inMinutes > 0) {
      return '${difference.inMinutes}分钟前';
    } else {
      return '刚刚';
    }
  }
}
```

## 表单页面

### 添加餐厅表单

```dart
class AddRestaurantForm extends StatefulWidget {
  const AddRestaurantForm({super.key});

  @override
  State<AddRestaurantForm> createState() => _AddRestaurantFormState();
}

class _AddRestaurantFormState extends State<AddRestaurantForm> {
  final _formKey = GlobalKey<FormState>();
  final _nameController = TextEditingController();
  final _photoController = TextEditingController();

  String? _selectedCategory;
  String? _selectedCity;
  int? _selectedPrice;

  bool _isLoading = false;

  final List<String> _categories = ['中餐', '西餐', '日料', '快餐', '咖啡厅'];
  final List<String> _cities = ['北京', '上海', '深圳', '广州'];

  @override
  void dispose() {
    _nameController.dispose();
    _photoController.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    if (!_formKey.currentState!.validate()) return;

    setState(() => _isLoading = true);

    try {
      final restaurant = Restaurant(
        name: _nameController.text.trim(),
        category: _selectedCategory,
        city: _selectedCity,
        price: _selectedPrice,
        photo: _photoController.text.trim().isNotEmpty
            ? _photoController.text.trim()
            : null,
      );

      // 这里应该调用餐厅服务添加餐厅
      // await _restaurantService.addRestaurant(restaurant);

      if (mounted) {
        Navigator.of(context).pop();
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('餐厅添加成功')),
        );
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('添加失败: $e')),
        );
      }
    } finally {
      if (mounted) {
        setState(() => _isLoading = false);
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('添加餐厅'),
      ),
      body: Form(
        key: _formKey,
        child: ListView(
          padding: const EdgeInsets.all(16),
          children: [
            // 餐厅名称
            TextFormField(
              controller: _nameController,
              decoration: const InputDecoration(
                labelText: '餐厅名称 *',
                border: OutlineInputBorder(),
              ),
              validator: (value) {
                if (value == null || value.trim().isEmpty) {
                  return '请输入餐厅名称';
                }
                return null;
              },
            ),

            const SizedBox(height: 16),

            // 类别
            DropdownButtonFormField<String>(
              value: _selectedCategory,
              decoration: const InputDecoration(
                labelText: '类别 *',
                border: OutlineInputBorder(),
              ),
              items: _categories.map((category) {
                return DropdownMenuItem(
                  value: category,
                  child: Text(category),
                );
              }).toList(),
              validator: (value) {
                if (value == null) {
                  return '请选择类别';
                }
                return null;
              },
              onChanged: (value) {
                setState(() => _selectedCategory = value);
              },
            ),

            const SizedBox(height: 16),

            // 城市
            DropdownButtonFormField<String>(
              value: _selectedCity,
              decoration: const InputDecoration(
                labelText: '城市 *',
                border: OutlineInputBorder(),
              ),
              items: _cities.map((city) {
                return DropdownMenuItem(
                  value: city,
                  child: Text(city),
                );
              }).toList(),
              validator: (value) {
                if (value == null) {
                  return '请选择城市';
                }
                return null;
              },
              onChanged: (value) {
                setState(() => _selectedCity = value);
              },
            ),

            const SizedBox(height: 16),

            // 价格等级
            DropdownButtonFormField<int>(
              value: _selectedPrice,
              decoration: const InputDecoration(
                labelText: '价格等级',
                border: OutlineInputBorder(),
              ),
              items: [
                const DropdownMenuItem(value: 1, child: Text('💰 经济实惠')),
                const DropdownMenuItem(value: 2, child: Text('💰💰 中等价格')),
                const DropdownMenuItem(value: 3, child: Text('💰💰💰 高档消费')),
              ],
              onChanged: (value) {
                setState(() => _selectedPrice = value);
              },
            ),

            const SizedBox(height: 16),

            // 照片URL
            TextFormField(
              controller: _photoController,
              decoration: const InputDecoration(
                labelText: '照片 URL',
                border: OutlineInputBorder(),
                hintText: 'https://example.com/photo.jpg',
              ),
            ),

            const SizedBox(height: 24),

            // 提交按钮
            ElevatedButton(
              onPressed: _isLoading ? null : _submit,
              style: ElevatedButton.styleFrom(
                padding: const EdgeInsets.symmetric(vertical: 16),
              ),
              child: _isLoading
                  ? const CircularProgressIndicator()
                  : const Text('添加餐厅'),
            ),
          ],
        ),
      ),
    );
  }
}
```

## 主题和样式

### 应用主题配置

```dart
class AppTheme {
  static ThemeData get lightTheme {
    return ThemeData(
      primaryColor: const Color(0xFF4285F4),
      colorScheme: ColorScheme.fromSeed(
        seedColor: const Color(0xFF4285F4),
        brightness: Brightness.light,
      ),
      appBarTheme: const AppBarTheme(
        backgroundColor: Color(0xFF4285F4),
        foregroundColor: Colors.white,
        elevation: 2,
      ),
      cardTheme: CardTheme(
        elevation: 2,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(12),
        ),
      ),
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 12),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(8),
          ),
        ),
      ),
      inputDecorationTheme: InputDecorationTheme(
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(8),
        ),
        contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 16),
      ),
    );
  }

  static ThemeData get darkTheme {
    return ThemeData(
      brightness: Brightness.dark,
      primaryColor: const Color(0xFF4285F4),
      colorScheme: ColorScheme.fromSeed(
        seedColor: const Color(0xFF4285F4),
        brightness: Brightness.dark,
      ),
      cardTheme: CardTheme(
        elevation: 2,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(12),
        ),
      ),
    );
  }
}
```

## 总结与检查清单

通过本章的学习，你应该已经：

- [ ] 实现了完整的应用导航结构
- [ ] 创建了美观的餐厅列表和详情页面
- [ ] 实现了筛选和排序功能
- [ ] 创建了评论显示和添加功能
- [ ] 设计了表单页面用于数据输入
- [ ] 配置了应用主题和样式

## 完整教程总结

恭喜你完成了整个 Firebase Firestore 教程系列！通过这 8 章的学习，你已经掌握了：

1. **项目设置与 Firebase 配置** - 学会了配置 Firebase 项目和初始化应用
2. **数据模型与实体类** - 理解了如何设计和实现数据模型
3. **Firestore 基本 CRUD 操作** - 掌握了基本的数据库操作
4. **认证集成与安全规则** - 学会了用户认证和数据安全
5. **实时数据流与监听器** - 实现了实时数据更新
6. **查询筛选与索引** - 学会了高级查询和性能优化
7. **评论子集合与事务** - 掌握了复杂数据关系和事务处理
8. **UI 组件与导航** - 创建了完整的用户界面

现在你可以独立开发使用 Firebase Firestore 的 Flutter 应用了！继续探索和实践，你会发现更多高级特性和最佳实践。
