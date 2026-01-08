# Firebase Firestore 完整教程系列

本教程系列基于 FriendlyEats 示例项目，全面讲解如何使用 Firebase Firestore 构建 Flutter 餐厅推荐应用。通过实际项目代码，学习 Firestore 的各项核心功能和最佳实践。

## 目标读者

- Flutter 开发者希望学习 Firebase Firestore
- 对 NoSQL 数据库感兴趣的开发者
- 希望构建实时数据应用的开发者

## 学习目标

完成本教程后，你将能够：

- 配置 Firebase 项目并集成到 Flutter 应用
- 设计和实现 Firestore 数据模型
- 实现完整的 CRUD 操作
- 处理实时数据更新
- 实现用户认证和安全规则
- 使用查询、筛选和索引优化性能
- 处理子集合和事务操作

## 教程章节

1. [项目设置与 Firebase 配置](chapter-01-setup.md)
2. [数据模型与实体类](chapter-02-data-model.md)
3. [Firestore 基本 CRUD 操作](chapter-03-crud-operations.md)
4. [认证集成与安全规则](chapter-04-auth-integration.md)
5. [实时数据流与监听器](chapter-05-real-time-streaming.md)
6. [查询筛选与索引](chapter-06-filtering-querying.md)
7. [评论子集合与事务](chapter-07-reviews-subcollections.md)
8. [UI 组件与导航](chapter-08-ui-navigation.md)

## 项目概述

FriendlyEats 是一个餐厅推荐应用，展示了 Firestore 在实际应用中的完整使用场景：

- **餐厅数据管理**：添加、查询和筛选餐厅信息
- **实时评分系统**：用户可以为餐厅添加评论和评分
- **多维度筛选**：按类别、城市、价格等条件筛选餐厅
- **用户认证**：使用 Firebase Authentication 进行匿名登录

## 技术栈

- Flutter 3.0+
- Firebase Core 2.24.2
- Cloud Firestore 4.13.6
- Firebase Auth 4.15.3

## 开始学习

从 [第 1 章：项目设置与 Firebase 配置](chapter-01-setup.md) 开始，按照章节顺序逐步学习。每章都包含详细的代码示例和实践指导。
