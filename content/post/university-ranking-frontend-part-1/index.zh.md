---
title: "University Ranking Frontend 深度分析（第一部分）：导航架构与搜索实现"
date: 2025-12-07
draft: false
tags: ["React Native", "导航", "搜索", "Vibe Coding", "实现"]
categories: ["Building an App using vibe coding"]
description: "我们对 University Ranking Frontend 代码的深度分析第一部分。探索 App.js 中的导航编排，实现具有实时过滤和底部工作表模态框的 SearchScreen，并构建大学资料的 DetailPage。"
---

这是我们 University Ranking Frontend 代码深度分析的第一部分。如果你还没有阅读 {{< ref "university-ranking-frontend" >}} 介绍，我建议从那里开始。

## 概述

在第一部分，我们将探索：
1. **导航编排** - 如何在 `App.js` 中编排底部标签和堆栈导航器
2. **SearchScreen 实现** - 用实时过滤构建主搜索界面
3. **DetailPage 构造** - 显示包含动态内容的全面大学资料
4. **API 集成** - 用 axios 获取数据并处理异步操作

## 架构课程：我们为什么使用这种导航模式

许多开发者会过度思考导航。University Ranking Frontend 使用一个简单但强大的模式：

```
底部标签
├── 首页（搜索）
│   └── 堆栈导航器
│       ├── SearchScreen
│       ├── DetailPage
│       └── UniversitySourceRankingsPage
├── 学科排名
│   └── 堆栈导航器
│       ├── SubjectRankingsPage
│       ├── RankingDetailPage
│       └── DetailPage
├── 收藏
│   └── 堆栈导航器
│       └── FavoritesScreen
└── 我的（个人资料）
    └── 堆栈导航器
        └── MeScreen
```

**为什么这样有效：**
- **底部标签**在主应用部分之间提供持久导航
- **每个标签内的堆栈导航器**启用推送/弹出转换以显示详细视图
- **共享屏幕**（DetailPage 出现在多个堆栈中）在不同上下文中重用组件

这是 **Vibe Coding 的实际应用**：与其发明自定义导航系统，我们使用经证明的 React Navigation 模式，AI 可以快速建议和改进。

## App.js：导航编排器

让我们检查根 App.js 文件，它设置了整个导航层次结构：

```javascript
import React, { useState, useEffect } from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { BottomSheetModalProvider } from '@gorhom/bottom-sheet';
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

// 初始化 React Query 客户端
const queryClient = new QueryClient({
	defaultOptions: {
		queries: {
			staleTime: 5 * 60 * 1000, // 5 分钟
			cacheTime: 10 * 60 * 1000, // 10 分钟
			retry: 2,
			refetchOnWindowFocus: false,
		},
	},
});

const Stack = createNativeStackNavigator();
const Tab = createBottomTabNavigator();
```

关键配置决策：

1. **Stale Time（5 分钟）**：5 分钟后，缓存数据被视为陈旧，React Query 将在后台刷新
2. **Cache Time（10 分钟）**：在垃圾收集前保持缓存数据 10 分钟
3. **Retry（2）**：在显示错误前自动重试失败的请求两次
4. **refetchOnWindowFocus（false）**：应用重新获得焦点时不重新获取（为了更好的用户体验）

这个配置是务实的：它平衡了响应性和网络效率。

### Home 堆栈（SearchScreen）

Home 堆栈是主导航入口点：

```javascript
function HomeScreen() {
	const { theme } = useTheme();

	return (
		<Stack.Navigator
			screenOptions={{
				headerStyle: {
					backgroundColor: theme.surface,
				},
				headerTintColor: theme.text,
				headerTitleStyle: {
					color: theme.text,
				},
			}}
		>
			<Stack.Screen
				name="SearchScreen"
				component={SearchScreen}
				options={{ headerShown: false }}
			/>
			<Stack.Screen
				name="DetailPage"
				component={UniversityDetail}
				options={({ route }) => ({
					title: route.params.name || i18n.t('university_details'),
				})}
			/>
			<Stack.Screen
				name="UniversitySourceRankingsPage"
				component={UniversitySourceRankingsPage}
				options={({ route }) => ({
					title: route.params.universityName + ' ' + formatSourceName(route.params.source),
				})}
			/>
		</Stack.Navigator>
	);
}
```

**关键模式**：头部样式是主题感知的。每当用户切换深色模式时，这些颜色实时更新，因为我们从 `useTheme()` hook 中读取。

### Context Provider 组合

App.js 用 context providers 包装整个导航树：

```javascript
function AppRoot() {
	return (
		<QueryClientProvider client={queryClient}>
			<ThemeProvider>
				<LanguageProvider>
					<UniversityTranslationsProvider>
						<RankingsProvider>
							<AppContentInner />
						</RankingsProvider>
					</UniversityTranslationsProvider>
				</LanguageProvider>
			</ThemeProvider>
		</QueryClientProvider>
	);
}
```

**顺序很重要：**
1. **QueryClientProvider**（最外层）- 管理所有服务器状态
2. **ThemeProvider** - 为所有组件提供配色方案
3. **LanguageProvider** - 提供语言和翻译函数
4. **UniversityTranslationsProvider** - 提供缓存的大学名称翻译
5. **RankingsProvider** - 为过滤器提供排名选项

每个 provider 都是独立的，可以单独测试。

## SearchScreen：实时大学搜索

SearchScreen 是应用中最复杂的组件。它实现了：
- 带防抖的实时搜索
- 带底部工作表的过滤器选择
- 动画加载旋转器
- 带平台特定调整的响应式布局

### 用 Effects 进行数据获取

```javascript
export default function SearchScreen() {
    const [query, setQuery] = useState('');
    const [country, setCountry] = useState(null);
    const [sortCredit, setSortCredit] = useState('QS_World_University_Rankings');
    const [results, setResults] = useState([]);
    const [isLoading, setIsLoading] = useState(false);
    
    // 当过滤器改变时获取大学
    useEffect(() => {
        let isMounted = true;

        const fetchData = async () => {
            if (!isMounted) return;
            
            setIsLoading(true);
            try {
                const data = await searchUniversities({
                    query,
                    country,
                    sort_credit: sortCredit
                });
                
                if (!isMounted) return; // 如果卸载则不更新
                
                const cleaned = data.map(u => ({
                    ...u,
                    country: u.country || '',
                    city: u.city || ''
                }));
                setResults(cleaned);
            } catch (error) {
                console.error('Error fetching universities:', error);
                if (isMounted) setResults([]);
            } finally {
                if (isMounted) setIsLoading(false);
            }
        };

        fetchData();

        return () => {
            isMounted = false; // 防止在卸载的组件上更新
        };
    }, [query, country, sortCredit]); // 当过滤器改变时重新运行
}
```

**Vibe Coding 模式**：`isMounted` 标志防止内存泄漏和竞态条件。AI 建议了这个模式，因为：
- 导航可以在异步操作完成前卸载屏幕
- 没有 `isMounted`，在卸载的组件上更新状态会导致警告
- 这个模式对于 React hooks 是习用的

### 用底部工作表模态框进行过滤

SearchScreen 使用底部工作表模态框进行国家和排名选择：

```javascript
import { BottomSheetModal, BottomSheetScrollView, BottomSheetBackdrop } from '@gorhom/bottom-sheet';

const countrySheetRef = useRef(null);
const rankingSheetRef = useRef(null);
const snapPoints = useMemo(() => ['60%', '75%', '90%'], []);

// 打开国家过滤器
const handleCountryPress = useCallback(() => {
    setCountrySearchQuery('');
    rankingSheetRef.current?.dismiss(); // 关闭其他工作表
    countrySheetRef.current?.present();
}, []);

// 当用户输入时过滤国家
const filteredCountryItems = useMemo(() => {
    if (!countrySearchQuery) return countryItems;
    return countryItems.filter(item =>
        item.label.toLowerCase().includes(countrySearchQuery.toLowerCase())
    );
}, [countryItems, countrySearchQuery]);
```

**关键见解**：底部工作表提供原生般的用户体验，其中过滤器易于访问但不隐藏主要内容。`snapPoints` 允许用户将工作表拖动到不同的高度。

### 搜索结果显示

搜索结果显示在 FlatList 中，用于高效渲染大型列表：

```javascript
<FlatList
    data={results}
    keyExtractor={(item) => item.normalized_name}
    renderItem={({ item }) => (
        <TouchableOpacity
            onPress={() => navigation.navigate('DetailPage', {
                normalized_name: item.normalized_name,
                name: item.name,
                country: item.country,
            })}
        >
            <View style={styles.resultItem}>
                <Text style={styles.resultName}>{item.name}</Text>
                <Text style={styles.resultLocation}>
                    {item.city}, {item.country}
                </Text>
            </View>
        </TouchableOpacity>
    )}
    ListEmptyComponent={() => (
        <Text style={styles.emptyText}>
            {isLoading ? i18n.t('searching') : i18n.t('no_results')}
        </Text>
    )}
/>
```

**模式**：使用 `normalized_name` 作为列表键和传递到 DetailPage 的标识符，确保整个应用的一致性。

## DetailPage：全面的大学资料

DetailPage 显示：
- 大学徽标和基本信息
- 跨多个来源的排名
- 大学描述（必要时动态翻译）
- 收藏/取消收藏功能
- 链接到详细排名细节

### 异步数据加载

```javascript
export default function GetUniversityDetail(props) {
    const normalized_name = props.route.params.normalized_name;
    const [university, setUniversity] = useState(null);
    const [loading, setLoading] = useState(true);
    const [translatedBlurb, setTranslatedBlurb] = useState('');
    const [translationLoading, setTranslationLoading] = useState(false);
    
    const { isChinese } = useLanguage();

    // 获取大学详情
    useEffect(() => {
        if (normalized_name) {
            getUniversityDetails(normalized_name)
                .then(data => {
                    if (data.notFound) {
                        navigation.goBack();
                    } else {
                        setUniversity(data);
                        setLoading(false);
                        checkIfFavorite(data);
                    }
                })
                .catch(err => {
                    console.error('Error fetching details:', err);
                    setLoading(false);
                });
        }
    }, [normalized_name]);

    // 当语言改变时翻译描述
    useEffect(() => {
        let isMounted = true;

        const translateBlurbIfNeeded = async () => {
            if (university && university.blurb && isChinese) {
                setTranslationLoading(true);
                try {
                    const translated = await translateText(university.blurb, 'en', 'zh-Hans');
                    if (isMounted) {
                        setTranslatedBlurb(translated);
                    }
                } catch (error) {
                    console.error('Translation failed:', error);
                }
            } else {
                if (isMounted) setTranslatedBlurb('');
            }
        };

        translateBlurbIfNeeded();
        return () => { isMounted = false; };
    }, [university, isChinese]);
}
```

**关键模式**：翻译作为副作用处理，当语言上下文改变时。这保持了翻译逻辑与组件渲染分离。

### 用 AsyncStorage 管理收藏

```javascript
const checkIfFavorite = async (universityData) => {
    try {
        const stored = await AsyncStorage.getItem('favoriteUniversities');
        if (stored) {
            const favorites = JSON.parse(stored);
            const isCurrentFavorite = favorites.some(fav =>
                fav.normalized_name === universityData.normalized_name ||
                fav.id === universityData.id
            );
            setIsFavorite(isCurrentFavorite);
        }
    } catch (error) {
        console.error('Error checking favorites:', error);
    }
};

const toggleFavorite = async () => {
    if (!university) return;

    setFavoriteLoading(true);
    try {
        const stored = await AsyncStorage.getItem('favoriteUniversities');
        let favorites = stored ? JSON.parse(stored) : [];

        const favoriteData = {
            id: university.id || university.normalized_name,
            normalized_name: university.normalized_name,
            name: university.name,
            country: university.country,
            city: university.city,
            photo: university.photo
        };

        if (isFavorite) {
            // 从收藏中删除
            favorites = favorites.filter(fav =>
                fav.normalized_name !== university.normalized_name &&
                fav.id !== university.id
            );
            setIsFavorite(false);
        } else {
            // 添加到收藏
            favorites.push(favoriteData);
            setIsFavorite(true);
        }

        await AsyncStorage.setItem('favoriteUniversities', JSON.stringify(favorites));
        await updateFavoriteCount(favorites.length);
    } catch (error) {
        console.error('Error updating favorites:', error);
    } finally {
        setFavoriteLoading(false);
    }
};
```

**Vibe Coding 见解**：与其向后端同步收藏，我们用 AsyncStorage 本地存储它们。这提供了：
- 即时反馈（无网络延迟）
- 离线工作
- 减少服务器负载

`favoriteLoading` 状态确保 UI 在切换操作期间显示加载状态。

## API 集成：lib/api.js 层

API 层将 API 调用与组件分离：

```javascript
import { API_BASE_URL } from '../constants/config';
import axios from 'axios';

const rawApi = {
    async searchUniversities({ query, country, city, sort_credit }) {
        const response = await axios.get(`${API_BASE_URL}/universities/filter`, {
            params: { query, country, city, sort_credit }
        });
        return response.data;
    },

    async getUniversityDetails(normalizedName) {
        const response = await axios.get(`${API_BASE_URL}/universities/${normalizedName}`);
        if (response.status !== 200) {
            throw new Error(`Failed to fetch: ${response.status}`);
        }
        return response.data;
    },

    async translateText(text, from, to) {
        const response = await axios.post(`${API_BASE_URL}/translate`, {
            text,
            from_lang: from,
            to_lang: to
        });
        return response.data.translatedText;
    }
};

// 导出函数供直接使用
export const searchUniversities = rawApi.searchUniversities;
export const getUniversityDetails = rawApi.getUniversityDetails;
export const translateText = rawApi.translateText;
```

**这个模式的好处：**
- **可测试性**：易于为单元测试模拟 `rawApi`
- **可重用性**：API 函数可以直接使用或用 React Query 包装
- **可维护性**：所有 API 调用在一个地方

## 从 Vibe Coding 得到的设计见解

### 1. 实时搜索比防抖更简单

最初，防抖搜索似乎是必需的以减少 API 调用。然而，后端过滤端点很快，网络很可靠。所以我们跳过防抖并让用户看到即时结果。**这是 Vibe Coding**：务实的选择优于理论上的优化。

### 2. 底部工作表 > 模态对话框

原生般的底部工作表提供比居中模态框更好的用户体验，因为：
- 用户可以看到工作表后的内容
- 拖动关闭感觉自然
- 屏幕空间使用高效

AI 建议 `@gorhom/bottom-sheet`，因为它提供平滑的动画和手势处理。

### 3. Context + AsyncStorage > 用于偏好的 Redux

用 AsyncStorage 和 React Context 本地管理收藏比以下更简单：
- Redux + Redux Persist
- 同步到后端
- 复杂的状态管理

Vibe Coding 帮助我们认识到并非所有内容都需要在全局状态中。

## 总结

在第一部分中，我们探索了：
- ✅ 导航如何在 App.js 中用底部标签和堆栈导航器编排
- ✅ 用实时过滤和底部工作表模态框构建 SearchScreen
- ✅ 实现具有异步数据加载和收藏管理的 DetailPage
- ✅ 为可重用性和可测试性构造 API 调用

在 {{< ref "university-ranking-frontend-part-2" >}} 中，我们将深入探讨：
- Context 实现细节（ThemeContext、LanguageContext、RankingsProvider）
- React Query 用于复杂数据管理的模式
- 主题感知 UI 的组件样式模式
- 平台特定的考虑因素

关键要点：**好的导航架构启用干净的组件设计**。通过在 App.js 中处理导航问题，单个屏幕可以专注于其特定职责——搜索、显示和管理数据。
