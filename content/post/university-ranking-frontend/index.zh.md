---
title: "用 Vibe Coding 构建 University Ranking Frontend：架构与设计策略"
date: 2025-12-07
draft: false
tags: ["React Native", "Vibe Coding", "移动开发", "前端", "Expo"]
categories: ["Building an App using vibe coding"]
description: "探索我们如何使用 Vibe Coding（AI 辅助开发）构建 University Ranking Frontend——一个注重实用选择和快速迭代的生产级 React Native 移动应用。"
---

## 介绍

University Ranking Frontend 是一个精细的 React Native 应用，展示了如何使用 **Vibe Coding** 构建生产级移动体验——一种开发哲学，其中 AI 协助指导架构决策、简化状态管理并加速功能实现。

与构建后端服务不同（需要谨慎的数据库设计和算法优化），前端开发通常关乎创建直观的用户体验和管理复杂的应用状态。在本系列中，我们将探索 Vibe Coding 如何将前端开发从劳动密集型过程转变为迭代的、AI 协助的协作，生成干净、可维护的代码。

## 什么是 Vibe Coding？

Vibe Coding 意味着**与 AI 协作构建软件**，利用机器学习模型来：

- **生成架构模式**，符合问题空间的需求
- **建议状态管理解决方案**，适配你的数据流
- **加速组件开发**，通过智能代码生成
- **验证设计模式**，在实现前识别潜在改进
- **快速迭代**功能，通过 AI 生成的模板代码和脚手架

对于 University Ranking Frontend，Vibe Coding 帮助我们：

1. **选择 React Native + Expo** 作为开发框架，基于对跨平台支持的需求（iOS、Android、Web）
2. **设计 Context API 系统**，用于主题、语言和排名管理，无需过度工程化
3. **构造 API 集成**，使用 React Query 进行高效的数据获取和缓存
4. **构建 UI 组件**，无缝适配深色模式和语言切换
5. **实现搜索过滤**，提供响应迅速的原生体验

## 项目概述

University Ranking Frontend 是一个移动优先的应用，允许用户：

- **搜索大学**，按名称、国家或城市实时筛选
- **查看详细排名**，跨多个排名系统（QS、US News 等）
- **切换主题**（浅色/深色模式），带持久化偏好
- **切换语言**（英文/中文），完全本地化 UI
- **保存收藏**，快速访问经常查看的大学
- **比较排名**，跨不同指标和来源

### 技术栈

**核心框架：**
- **React Native**：跨平台移动开发
- **Expo**：React Native 的开发平台和托管构建系统
- **@react-navigation**：导航库，支持底部标签和堆栈导航器

**状态管理：**
- **React Context API**：主题、语言和排名状态
- **@tanstack/react-query**：服务器状态管理，支持缓存和后台同步
- **AsyncStorage / Expo FileSystem**：本地持久化偏好和缓存

**UI 与样式：**
- **React Native 内置组件**：View、Text、FlatList、ScrollView 等
- **@gorhom/bottom-sheet**：原生底部工作表模态框
- **@expo/vector-icons**：图标库（Ionicons）

**国际化与 API：**
- **i18n-js**：英文/中文国际化支持
- **axios**：HTTP 客户端
- **必应翻译 API**：动态内容翻译（例如大学描述）

### 应用架构

```
┌─────────────────────────────────────────┐
│         App.js（根导航器）              │
│  - Query Client 配置                    │
│  - Context Providers 设置               │
└─────────────────────────────────────────┘
                    ↓
    ┌───────────────────────────────┐
    │   底部标签导航                 │
    │  ┌─────────────────────────┐  │
    │  │  搜索堆栈               │  │
    │  │  - SearchScreen         │  │
    │  │  - DetailPage           │  │
    │  └─────────────────────────┘  │
    │  ┌─────────────────────────┐  │
    │  │  排名堆栈               │  │
    │  │  - SubjectRankings      │  │
    │  │  - RankingDetailPage    │  │
    │  └─────────────────────────┘  │
    │  ┌─────────────────────────┐  │
    │  │  其他堆栈               │  │
    │  │  - 收藏                 │  │
    │  │  - 我的（个人资料）     │  │
    │  └─────────────────────────┘  │
    └───────────────────────────────┘
                    ↓
    ┌───────────────────────────────┐
    │   Context Providers           │
    │  ┌─────────────────────────┐  │
    │  │  ThemeProvider          │  │
    │  │  - 深色/浅色主题        │  │
    │  │  - 平台特定样式         │  │
    │  └─────────────────────────┘  │
    │  ┌─────────────────────────┐  │
    │  │  LanguageProvider       │  │
    │  │  - 英文/中文支持        │  │
    │  │  - 设备语言检测         │  │
    │  └─────────────────────────┘  │
    │  ┌─────────────────────────┐  │
    │  │  RankingsProvider       │  │
    │  │  - 学科排名             │  │
    │  │  - 缓存/同步            │  │
    │  └─────────────────────────┘  │
    └───────────────────────────────┘
                    ↓
    ┌───────────────────────────────┐
    │   API 集成层                   │
    │  - React Query hooks           │
    │  - 原始 axios 函数             │
    │  - 本地文件缓存                │
    └───────────────────────────────┘
```

## 使用 Vibe Coding 的构建策略

### 第一阶段：基础设施（导航与 Context 设置）

**挑战**：如何构造一个多屏幕应用，用户可以搜索、查看详情、比较排名和管理偏好？

**Vibe Coding 方法**：与其过度工程化复杂的状态管理解决方案，我们使用了：
- **底部标签导航**，用于主要部分（搜索、排名、收藏、我的）
- **每个标签内的堆栈导航器**，用于分层屏幕
- **React Context API**，用于在整个应用中访问的全局状态（主题、语言、排名）

这个务实的选择避免了 Redux/Mobx 的复杂性，同时保持了关注点的清晰分离。

### 第二阶段：状态管理（Context 模式）

**挑战**：如何在整个应用中管理主题切换、语言偏好和排名选项，而不进行 prop drilling？

**Vibe Coding 方法**：三个独立的 context providers，每个处理特定的关注点：

1. **ThemeContext**：管理深色/浅色模式，带平台特定的样式（Android/iOS/Web）
2. **LanguageContext**：处理语言切换，带设备语言检测
3. **RankingsProvider**：使用 React Query 获取排名选项，带本地缓存（原生上使用 FileSystem，Web 上使用 localStorage）

每个 context 都是独立的，可以单独测试。新功能可以接入现有 context 而不会造成中断。

### 第三阶段：API 与数据获取

**挑战**：如何高效地获取大学数据、缓存结果并保持 UI 响应性？

**Vibe Coding 方法**：我们分层了两种 API 模式：

1. **原始 API 函数**（`lib/api.js`）：基础 axios 调用，用于直接数据获取
2. **React Query hooks**：包装原始函数，带缓存、后台同步和自动重试逻辑

这种两层方法意味着：
- 简单组件可以使用原始 API 调用进行一次性请求
- 复杂屏幕使用 React Query 获得复杂的缓存策略
- 我们可以重构而无需触及屏幕代码

### 第四阶段：UI 实现

**挑战**：如何创建一个响应式 UI，在手机、平板和 Web 上都能工作，同时支持深色模式和多种语言？

**Vibe Coding 方法**：带有上下文感知样式的组件组合：
- **主题感知样式**：每个组件从 ThemeContext 读取颜色
- **语言感知文本**：所有 UI 字符串使用 i18n 实现即时语言切换
- **平台特定调整**：适配 iOS、Android 和 Web 的布局调整
- **底部工作表模态框**：用于过滤和选择，提供原生般的用户体验

## 关键见解

### 1. React Context 就足够了

许多开发者会为状态管理跳转到 Redux 或 Mobx。对于 University Ranking Frontend，React Context + hooks 足以胜任：
- 全局主题/语言状态
- 导航感知状态
- 懒加载的排名选项

我们避免了过度工程化——这是 Vibe Coding 原则之一。

### 2. React Query 改变数据管理

与其手动管理 API 请求状态（加载、错误、成功），React Query 处理：
- 请求去重
- 自动重试
- 陈旧数据失效
- 后台同步

这大大减少了样板代码并提高了可靠性。

### 3. Expo 简化跨平台开发

与其维护单独的 iOS/Android 原生项目，Expo 提供：
- 单一 JavaScript 代码库
- 托管构建（无需 Xcode/Android Studio）
- 空中更新能力
- 通过 React Native Web 的 Web 支持

这完美契合 Vibe Coding——让 AI 处理平台差异，而开发者专注于逻辑。

### 4. 本地化必须尽早规划

支持多种语言不是事后才想的。University Ranking Frontend 从第一天起就是双语的（英文/中文）：
- i18n-js 处理字符串翻译
- UniversityTranslationsContext 缓存大学名称翻译
- API 集成包括翻译端点（必应翻译）
- 设备语言自动检测，实现无缝用户体验

## 架构亮点

### 底部标签导航

主导航结构使用 `@react-navigation/bottom-tabs`：

```javascript
<NavigationContainer>
  <Tab.Navigator
    screenOptions={({ route }) => ({
      tabBarIcon: ({ focused, color, size }) => {
        // 基于路由的动态图标选择
      },
    })}
  >
    <Tab.Screen name="Search" component={SearchNavigator} />
    <Tab.Screen name="Rankings" component={RankingsNavigator} />
    <Tab.Screen name="Favorites" component={FavoritesNavigator} />
    <Tab.Screen name="Me" component={MeNavigator} />
  </Tab.Navigator>
</NavigationContainer>
```

每个标签都有自己的堆栈导航器用于详细视图。

### Context Provider 组合

在 `App.js` 中，contexts 被嵌套以为所有屏幕提供数据：

```javascript
<QueryClientProvider client={queryClient}>
  <ThemeProvider>
    <LanguageProvider>
      <UniversityTranslationsProvider>
        <RankingsProvider>
          <NavigationContainer>
            {/* 导航结构 */}
          </NavigationContainer>
        </RankingsProvider>
      </UniversityTranslationsProvider>
    </LanguageProvider>
  </ThemeProvider>
</QueryClientProvider>
```

这个组合顺序很重要：ThemeProvider 和 LanguageProvider 应该包装一切，因为它们提供基础样式。

## 接下来是什么

在 {{< ref "university-ranking-frontend-part-1" >}} 中，我们将深入探讨：
- 导航编排如何在 `App.js` 中工作
- 用实时过滤和底部工作表模态框构建 SearchScreen
- 实现 DetailPage，带懒加载图像和动态翻译
- 保持应用响应性的导航模式

在 {{< ref "university-ranking-frontend-part-2" >}} 中，我们将探索：
- Context 实现细节（ThemeContext、LanguageContext、RankingsProvider）
- React Query 高效数据获取的模式
- 主题感知 UI 的组件架构
- 处理平台特定行为（iOS/Android/Web）

---

## 总结

University Ranking Frontend 证明了构建精细的移动应用不需要复杂的框架。通过 Vibe Coding，我们利用 AI 来：

1. 选择正确的技术栈（React Native + Expo）
2. 设计可扩展的状态管理（Context + React Query）
3. 用最少的样板代码构建响应式、双语 UI
4. 创建在 iOS、Android 和 Web 上都能工作的生产就绪型应用

关键见解：**好的架构是务实的，而不是理论的**。Vibe Coding 帮助我们快速而自信地做出这些务实的选择。

在本系列的下一部分中，我们将剖析代码，展示每一部分如何组合在一起。
