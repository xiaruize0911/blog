---
title: "University Ranking Frontend 深度分析（第二部分）：状态管理、React Query 和主题架构"
date: 2025-12-07
draft: false
tags: ["React Native", "Context API", "React Query", "状态管理", "Vibe Coding"]
categories: ["Building an App using vibe coding"]
description: "我们对 University Ranking Frontend 代码深度分析的第二部分。探索 ThemeContext 实现、LanguageContext、RankingsProvider 与 React Query，以及平台特定的优化。"
---

这是我们 University Ranking Frontend 代码深度分析的第二部分。确保你已经阅读了 {{< ref "university-ranking-frontend" >}} 和 {{< ref "university-ranking-frontend-part-1" >}}。

## 概述

在第二部分，我们将探索：
1. **ThemeContext** - 实现带平台特定持久化的主题切换
2. **LanguageContext** - 管理国际化和语言偏好
3. **RankingsProvider** - 使用 React Query 进行后台数据同步
4. **React Query Hooks** - 构建可重用的数据获取模式
5. **平台特定优化** - 不同方式处理 iOS、Android 和 Web

## ThemeContext：主题管理

`ThemeContext` 管理整个应用的视觉主题——浅色模式、深色模式或自动（跟随系统偏好）。以下是完整的实现：

### 主题对象

```javascript
export const lightTheme = {
    background: Platform.OS == 'android' ? '#ffffff' : '#f8f9fa',
    surface: '#ffffff',
    surfaceSecondary: '#f1f3f5',
    primary: '#4a90e2',
    text: '#2c3e50',
    textSecondary: '#6c757d',
    border: '#e1e5e9',
    card: '#ffffff',
    input: '#ffffff',
    shadow: '#000',
};

export const darkTheme = {
    background: Platform.OS == 'android' ? '#121212' : '#121212',
    surface: '#1e1e1e',
    surfaceSecondary: '#434343ff',
    primary: '#4a90e2',
    text: '#ffffff',
    textSecondary: '#b0b0b0',
    border: '#333333',
    card: '#2a2a2a',
    input: '#2a2a2a',
    shadow: '#000',
};
```

**注意：** `background` 在 Android 和 iOS 之间不同。这是 **Vibe Coding 的实际应用**——当 AI 生成平台特定代码时，它经常发现手动编码可能错过的这种细微差别。

### 带持久化的主题提供者

```javascript
export const ThemeProvider = ({ children }) => {
    const systemColorScheme = useColorScheme();
    const [themeMode, setThemeMode] = useState('auto'); // 'light', 'dark', 或 'auto'
    const [isLoading, setIsLoading] = useState(true);

    // 确定深色模式是否应该活跃
    const isDarkMode = themeMode === 'auto' 
        ? systemColorScheme === 'dark' 
        : themeMode === 'dark';

    // 启动时加载保存的主题偏好
    useEffect(() => {
        let isMounted = true;

        const loadSavedThemePreference = async () => {
            try {
                if (Platform.OS === 'web') {
                    const themePref = window.localStorage.getItem('themePreference');
                    if (themePref && isMounted) {
                        setThemeMode(themePref);
                    }
                } else {
                    // 移动设备：使用文件系统
                    const profileFile = FileSystem.documentDirectory + 'user_profile.json';
                    const profileExists = await FileSystem.getInfoAsync(profileFile);
                    if (profileExists.exists) {
                        const profileContent = await FileSystem.readAsStringAsync(profileFile);
                        const profile = JSON.parse(profileContent);
                        if (profile.themePreference && isMounted) {
                            setThemeMode(profile.themePreference);
                        }
                    }
                }
            } catch (error) {
                console.error('Error loading theme:', error);
                if (isMounted) setThemeMode('auto');
            } finally {
                if (isMounted) setIsLoading(false);
            }
        };

        loadSavedThemePreference();
        return () => { isMounted = false; };
    }, []);

    const toggleTheme = async () => {
        // 循环通过：light → dark → auto → light
        const nextMode = themeMode === 'light' 
            ? 'dark' 
            : themeMode === 'dark' 
            ? 'auto' 
            : 'light';
        
        setThemeMode(nextMode);

        // 持久化到相应的存储
        try {
            if (Platform.OS === 'web') {
                window.localStorage.setItem('themePreference', nextMode);
            } else {
                const profileFile = FileSystem.documentDirectory + 'user_profile.json';
                let profile = {
                    themePreference: nextMode,
                    lastUpdated: new Date().toISOString()
                };

                const profileExists = await FileSystem.getInfoAsync(profileFile);
                if (profileExists.exists) {
                    const existingProfileContent = await FileSystem.readAsStringAsync(profileFile);
                    const existingProfile = JSON.parse(existingProfileContent);
                    profile = {
                        ...existingProfile,
                        themePreference: nextMode,
                        lastUpdated: new Date().toISOString()
                    };
                }
                
                await FileSystem.writeAsStringAsync(
                    profileFile, 
                    JSON.stringify(profile, null, 2)
                );
            }
        } catch (error) {
            console.error('Error saving theme:', error);
        }
    };

    const theme = isDarkMode ? darkTheme : lightTheme;

    return (
        <ThemeContext.Provider value={{ theme, isDarkMode, toggleTheme }}>
            {children}
        </ThemeContext.Provider>
    );
};
```

**关键模式：**
1. **平台检测** - 使用 `Platform.OS` 在 'web'、'ios'、'android' 之间分支
2. **存储抽象** - Web 使用 localStorage，移动设备使用文件系统
3. **IsMounted 模式** - 防止在卸载的组件上状态更新
4. **三态切换** - 循环 light → dark → auto 以获得更好的用户体验

### 使用主题 Hook

组件通过 hook 访问主题：

```javascript
function MyComponent() {
    const { theme, isDarkMode, toggleTheme } = useTheme();

    return (
        <View style={{ backgroundColor: theme.background }}>
            <Text style={{ color: theme.text }}>内容</Text>
            <TouchableOpacity onPress={toggleTheme}>
                <Text>{isDarkMode ? 'Light' : 'Dark'} 模式</Text>
            </TouchableOpacity>
        </View>
    );
}
```

这个模式使每个组件都能感知主题，无需 prop drilling。

## LanguageContext：国际化

`LanguageContext` 管理语言切换和设备语言检测：

```javascript
export const LanguageProvider = ({ children }) => {
    const [currentLanguage, setCurrentLanguage] = useState(i18n.locale);
    const [isLoading, setIsLoading] = useState(true);

    // 启动时加载保存的语言偏好
    useEffect(() => {
        let isMounted = true;

        const loadLanguagePreference = async () => {
            try {
                const savedLang = await loadSavedLanguagePreference();
                if (savedLang && (savedLang === 'en' || savedLang === 'zh') && isMounted) {
                    setCurrentLanguage(savedLang);
                    i18n.locale = savedLang;
                } else if (isMounted) {
                    // 回退：使用设备语言
                    const deviceLang = Localization.locale.startsWith('zh') ? 'zh' : 'en';
                    setCurrentLanguage(deviceLang);
                    i18n.locale = deviceLang;
                }
            } catch (error) {
                console.error('Error loading language:', error);
                if (isMounted) {
                    const deviceLang = Localization.locale.startsWith('zh') ? 'zh' : 'en';
                    setCurrentLanguage(deviceLang);
                    i18n.locale = deviceLang;
                }
            } finally {
                if (isMounted) setIsLoading(false);
            }
        };

        loadLanguagePreference();
        return () => { isMounted = false; };
    }, []);

    const switchLanguage = async (lang) => {
        if (lang === 'en' || lang === 'zh') {
            setCurrentLanguage(lang);
            i18n.locale = lang;
            await saveLanguagePreference(lang);
        }
    };

    const contextValue = {
        currentLanguage,
        isChinese: currentLanguage === 'zh',
        switchLanguage,
        isLoading,
    };

    return (
        <LanguageContext.Provider value={contextValue}>
            {children}
        </LanguageContext.Provider>
    );
};
```

**聪明的模式：** 与其暴露原始语言字符串，我们提供 `isChinese` 布尔值。这使条件渲染更简单：

```javascript
// 在组件中：
const { isChinese } = useLanguage();

{isChinese ? (
    <ChineseVersionComponent />
) : (
    <EnglishVersionComponent />
)}
```

## RankingsProvider：带缓存的 React Query

`RankingsProvider` 是结合 React Query 与本地缓存的复杂示例：

```javascript
export const RankingsProvider = ({ children }) => {
    // 使用 React Query 在后台获取数据
    const { data: freshData, error, isLoading } = useRankingOptions();
    const [rankingOptions, setRankingOptions] = useState([]);
    const [loading, setLoading] = useState(true);

    // 挂载时从本地缓存加载
    useEffect(() => {
        let isMounted = true;

        const loadCache = async () => {
            try {
                let cachedData = null;

                if (Platform.OS === 'web') {
                    const cached = window.localStorage.getItem('subjectRankingOptions');
                    if (cached) {
                        cachedData = JSON.parse(cached);
                    }
                } else {
                    const cacheFile = FileSystem.documentDirectory + 'subject_ranking_options.json';
                    const info = await FileSystem.getInfoAsync(cacheFile);
                    if (info.exists) {
                        const cached = await FileSystem.readAsStringAsync(cacheFile);
                        cachedData = JSON.parse(cached);
                    }
                }

                if (isMounted && cachedData && Array.isArray(cachedData)) {
                    setRankingOptions(cachedData);
                }
                if (isMounted) setLoading(false);
            } catch (e) {
                console.error('Error loading cache:', e);
                if (isMounted) setLoading(false);
            }
        };

        loadCache();
        return () => { isMounted = false; };
    }, []);

    // 当来自 React Query 的新数据到达时更新缓存
    useEffect(() => {
        let isMounted = true;

        const updateCache = async () => {
            if (!isMounted || !freshData || !Array.isArray(freshData) || freshData.length === 0) {
                return;
            }

            setRankingOptions(freshData);
            setLoading(false);

            try {
                if (Platform.OS === 'web') {
                    window.localStorage.setItem('subjectRankingOptions', JSON.stringify(freshData));
                } else {
                    const cacheFile = FileSystem.documentDirectory + 'subject_ranking_options.json';
                    await FileSystem.writeAsStringAsync(cacheFile, JSON.stringify(freshData));
                }
            } catch (e) {
                console.error('Error saving cache:', e);
            }
        };

        if (!isLoading && freshData) {
            updateCache();
        }

        return () => { isMounted = false; };
    }, [freshData, isLoading]);

    return (
        <RankingsContext.Provider value={{
            rankingOptions: rankingOptions || [],
            loading,
            error
        }}>
            {children}
        </RankingsContext.Provider>
    );
};
```

**架构见解：** 这演示了 **两层缓存策略**：
1. **本地缓存**（立即显示）- 来自文件系统/localStorage
2. **React Query 缓存**（在后台更新）- 来自 API 服务器

当应用启动时：
1. 本地缓存立即加载
2. React Query 在后台获取新数据
3. 当新数据到达时，本地缓存被更新
4. 组件获得新数据而无延迟

这是 **Vibe Coding 的卓越表现**：AI 协助的方法认识到立即显示陈旧数据比在获取时显示什么都不显示更好。

## React Query Hooks：可重用模式

在 `lib/api.js` 中，我们为不同的数据类型定义 React Query hooks：

```javascript
export const useRankingOptions = (options = {}) => {
    return useQuery({
        queryKey: ['rankingOptions'],
        queryFn: rawApi.getRankingOptions,
        staleTime: 30 * 60 * 1000, // 30 分钟
        ...options
    });
};

export const useCountries = (options = {}) => {
    return useQuery({
        queryKey: ['countries'],
        queryFn: rawApi.getCountries,
        staleTime: 60 * 60 * 1000, // 1 小时
        ...options
    });
};

export const useUniversityDetails = (normalizedName, options = {}) => {
    return useQuery({
        queryKey: ['universityDetails', normalizedName],
        queryFn: () => rawApi.getUniversityDetails(normalizedName),
        staleTime: 10 * 60 * 1000, // 10 分钟
        enabled: !!normalizedName, // 仅在我们有名称时获取
        ...options
    });
};

export const useSearchUniversities = ({ query, country, city, sort_credit }, options = {}) => {
    return useQuery({
        queryKey: ['searchUniversities', { query, country, city, sort_credit }],
        queryFn: () => rawApi.searchUniversities({ query, country, city, sort_credit }),
        staleTime: 2 * 60 * 1000, // 2 分钟
        enabled: !!(query || country || city || sort_credit), // 仅在参数存在时运行
        ...options
    });
};
```

**关键模式：**

1. **Stale times 因数据类型而异**：
   - 排名选项：30 分钟（很少改变）
   - 国家：1 小时（静态数据）
   - 大学详情：10 分钟（偶尔改变）
   - 搜索结果：2 分钟（用户驱动）

2. **通过 `enabled` 进行条件获取**：
   ```javascript
   enabled: !!normalizedName  // 在我们有数据前不获取
   ```

3. **查询密钥结构**启用自动缓存管理：
   ```javascript
   queryKey: ['searchUniversities', { query, country, city, sort_credit }]
   // 对任何参数的更改自动失效缓存
   ```

### 使用 React Query Hooks

组件可以使用这些 hooks 进行复杂的数据管理：

```javascript
function RankingDetailComponent({ universityName, source }) {
    // 自动管理加载、错误和数据状态
    const { data, isLoading, error } = useUniversityRankingsBySource(
        universityName, 
        source
    );

    if (isLoading) return <ActivityIndicator />;
    if (error) return <Text>Error loading rankings</Text>;

    return (
        <FlatList
            data={data}
            renderItem={({ item }) => <RankingItem item={item} />}
            keyExtractor={(item) => item.id}
        />
    );
}
```

## 平台特定优化

应用以不同的方式处理 iOS、Android 和 Web：

### 存储策略

**Web：**
```javascript
window.localStorage.setItem('key', JSON.stringify(value));
```

**移动设备：**
```javascript
import * as FileSystem from 'expo-file-system';
const filePath = FileSystem.documentDirectory + 'myfile.json';
await FileSystem.writeAsStringAsync(filePath, JSON.stringify(value));
```

**为什么不同？** Web 没有 `FileSystem`，移动设备 localStorage 有大小限制。

### SafeAreaView 和 Insets

```javascript
function AppContentInner() {
    const insets = useSafeAreaInsets();
    
    const paddingTop = Platform.OS === 'android' ? insets.top : 0;
    const paddingBottom = Platform.OS === 'android' ? insets.bottom : 0;

    return (
        <GestureHandlerRootView 
            style={{ 
                flex: 1, 
                paddingTop, 
                paddingBottom,
                backgroundColor: theme.surface 
            }}
        >
            {/* 内容 */}
        </GestureHandlerRootView>
    );
}
```

**原理：** Android 尊重安全区域，但 iOS 不需要手动填充，因为 SafeAreaProvider 处理它。

### 头部高度调整

```javascript
const tabBarStyle = {
    backgroundColor: theme.surface,
    borderTopColor: theme.border,
    borderTopWidth: 1,
    paddingBottom: 0,
    height: 85,
    elevation: 0,           // Android 阴影
    shadowColor: 'transparent', // iOS 阴影
    shadowOpacity: 0,
    shadowOffset: { height: 0 },
    shadowRadius: 0,
};
```

iOS（shadow*）和 Android（elevation）不同的阴影属性。

## Vibe Coding 见解

### 1. Context + Hooks > Props

通过 props 传递主题/语言需要在每个组件中梳理它们。Context + hooks 消除 prop drilling：

```javascript
// ❌ 没有 context
<App theme={theme} language={language}>
  <Navigation theme={theme} language={language}>
    <Screen theme={theme} language={language}>
      <Component theme={theme} language={language} />
    </Screen>
  </Navigation>
</App>

// ✅ 使用 context
<ThemeProvider>
  <LanguageProvider>
    <App>
      {/* 任何组件都可以 useTheme() 和 useLanguage() */}
    </App>
  </LanguageProvider>
</ThemeProvider>
```

### 2. React Query 启用离线优先

通过适当的 React Query 配置，屏幕可以立即显示缓存数据，然后在网络可用时在后台更新。这优于显示加载旋转器。

### 3. 平台检测很务实

与其尝试为所有平台创建一个解决方案，检测平台并应用特定逻辑通常比抽象层更干净。

## 总结

在第二部分，我们探索了：
- ✅ 用于主题切换和平台特定持久化的 ThemeContext
- ✅ 用于智能语言管理的 LanguageContext
- ✅ 使用 React Query + 本地缓存的 RankingsProvider
- ✅ 用于不同数据类型的可重用 React Query hooks
- ✅ iOS、Android 和 Web 的平台特定优化

关键要点：**将关注点分离为 context providers 使每个组件都更简单**。通过在专用 contexts 中处理主题、语言和排名管理，SearchScreen 和 DetailPage 等屏幕可以纯粹专注于其业务逻辑。

这是 Vibe Coding 的实践：使用 AI 协助设计干净、务实的解决方案，在整个平台上无缝工作，同时保持代码清晰度。
