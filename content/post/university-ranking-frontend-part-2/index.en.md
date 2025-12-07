---
title: "University Ranking Frontend Deep Dive (Part 2): State Management, React Query, and Theme Architecture"
date: 2025-12-07
draft: false
tags: ["React Native", "Context API", "React Query", "State Management", "Vibe Coding"]
categories: ["Building an App using vibe coding"]
description: "Part 2 of our deep dive into the University Ranking Frontend. We explore ThemeContext implementation, LanguageContext, RankingsProvider with React Query, and platform-specific optimizations."
---

This is Part 2 of our University Ranking Frontend code deep dive. Make sure you've read {{< ref "university-ranking-frontend" >}} and {{< ref "university-ranking-frontend-part-1" >}} first.

## Overview

In Part 2, we'll explore:
1. **ThemeContext** - Implementing theme switching with platform-specific persistence
2. **LanguageContext** - Managing internationalization and language preferences
3. **RankingsProvider** - Using React Query for background data synchronization
4. **React Query Hooks** - Building reusable data fetching patterns
5. **Platform-Specific Optimizations** - Handling iOS, Android, and Web differently

## ThemeContext: Theme Management

The `ThemeContext` manages the entire app's visual theme—light mode, dark mode, or automatic (following system preference). Here's the complete implementation:

### Theme Objects

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

**Notice:** `background` differs between Android and iOS. This is **Vibe Coding at work**—when AI generates platform-specific code, it often discovers such nuances that manual coding might miss.

### Theme Provider with Persistence

```javascript
export const ThemeProvider = ({ children }) => {
    const systemColorScheme = useColorScheme();
    const [themeMode, setThemeMode] = useState('auto'); // 'light', 'dark', or 'auto'
    const [isLoading, setIsLoading] = useState(true);

    // Determine if dark mode should be active
    const isDarkMode = themeMode === 'auto' 
        ? systemColorScheme === 'dark' 
        : themeMode === 'dark';

    // Load saved theme preference on startup
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
                    // Mobile: use FileSystem
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
        // Cycle through: light → dark → auto → light
        const nextMode = themeMode === 'light' 
            ? 'dark' 
            : themeMode === 'dark' 
            ? 'auto' 
            : 'light';
        
        setThemeMode(nextMode);

        // Persist to appropriate storage
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

**Key patterns:**
1. **Platform detection** - Use `Platform.OS` to branch between 'web', 'ios', 'android'
2. **Storage abstraction** - Web uses localStorage, mobile uses FileSystem
3. **IsMounted pattern** - Prevents state updates on unmounted components
4. **Three-state toggle** - Cycles light → dark → auto for better UX

### Using the Theme Hook

Components access the theme via the hook:

```javascript
function MyComponent() {
    const { theme, isDarkMode, toggleTheme } = useTheme();

    return (
        <View style={{ backgroundColor: theme.background }}>
            <Text style={{ color: theme.text }}>Content</Text>
            <TouchableOpacity onPress={toggleTheme}>
                <Text>{isDarkMode ? 'Light' : 'Dark'} Mode</Text>
            </TouchableOpacity>
        </View>
    );
}
```

This pattern makes every component theme-aware without prop drilling.

## LanguageContext: Internationalization

The `LanguageContext` manages language switching and device locale detection:

```javascript
export const LanguageProvider = ({ children }) => {
    const [currentLanguage, setCurrentLanguage] = useState(i18n.locale);
    const [isLoading, setIsLoading] = useState(true);

    // Load saved language preference on startup
    useEffect(() => {
        let isMounted = true;

        const loadLanguagePreference = async () => {
            try {
                const savedLang = await loadSavedLanguagePreference();
                if (savedLang && (savedLang === 'en' || savedLang === 'zh') && isMounted) {
                    setCurrentLanguage(savedLang);
                    i18n.locale = savedLang;
                } else if (isMounted) {
                    // Fallback: use device language
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

**Smart pattern:** Instead of exposing raw language strings, we provide `isChinese` boolean. This makes conditional rendering simpler:

```javascript
// In components:
const { isChinese } = useLanguage();

{isChinese ? (
    <ChineseVersionComponent />
) : (
    <EnglishVersionComponent />
)}
```

## RankingsProvider: React Query with Caching

The `RankingsProvider` is a sophisticated example of combining React Query with local caching:

```javascript
export const RankingsProvider = ({ children }) => {
    // Use React Query to fetch data in background
    const { data: freshData, error, isLoading } = useRankingOptions();
    const [rankingOptions, setRankingOptions] = useState([]);
    const [loading, setLoading] = useState(true);

    // Load from local cache on mount
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

    // Update cache when fresh data arrives from React Query
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

**Architecture insight:** This demonstrates a **two-tier caching strategy**:
1. **Local cache** (displayed immediately) - From FileSystem/localStorage
2. **React Query cache** (updated in background) - From API server

When the app starts:
1. Local cache is loaded instantly
2. React Query fetches fresh data in background
3. When fresh data arrives, local cache is updated
4. Components get the fresh data without delay

This is **Vibe Coding excellence**: the AI-assisted approach recognizes that showing stale data instantly is better than showing nothing while fetching.

## React Query Hooks: Reusable Patterns

In `lib/api.js`, we define React Query hooks for different data types:

```javascript
export const useRankingOptions = (options = {}) => {
    return useQuery({
        queryKey: ['rankingOptions'],
        queryFn: rawApi.getRankingOptions,
        staleTime: 30 * 60 * 1000, // 30 minutes
        ...options
    });
};

export const useCountries = (options = {}) => {
    return useQuery({
        queryKey: ['countries'],
        queryFn: rawApi.getCountries,
        staleTime: 60 * 60 * 1000, // 1 hour
        ...options
    });
};

export const useUniversityDetails = (normalizedName, options = {}) => {
    return useQuery({
        queryKey: ['universityDetails', normalizedName],
        queryFn: () => rawApi.getUniversityDetails(normalizedName),
        staleTime: 10 * 60 * 1000, // 10 minutes
        enabled: !!normalizedName, // Only fetch if we have a name
        ...options
    });
};

export const useSearchUniversities = ({ query, country, city, sort_credit }, options = {}) => {
    return useQuery({
        queryKey: ['searchUniversities', { query, country, city, sort_credit }],
        queryFn: () => rawApi.searchUniversities({ query, country, city, sort_credit }),
        staleTime: 2 * 60 * 1000, // 2 minutes
        enabled: !!(query || country || city || sort_credit), // Only run if params exist
        ...options
    });
};
```

**Key patterns:**

1. **Stale times vary by data type**:
   - Ranking options: 30 min (rarely changes)
   - Countries: 1 hour (static data)
   - University details: 10 min (changes occasionally)
   - Search results: 2 min (user-driven)

2. **Conditional fetching** via `enabled`:
   ```javascript
   enabled: !!normalizedName  // Don't fetch until we have data
   ```

3. **Query key structure** enables automatic cache management:
   ```javascript
   queryKey: ['searchUniversities', { query, country, city, sort_credit }]
   // Changes to any parameter automatically invalidate the cache
   ```

### Using React Query Hooks

Components can use these hooks for sophisticated data management:

```javascript
function RankingDetailComponent({ universityName, source }) {
    // Automatically manages loading, error, and data states
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

## Platform-Specific Optimizations

The app handles iOS, Android, and Web differently:

### Storage Strategy

**Web:**
```javascript
window.localStorage.setItem('key', JSON.stringify(value));
```

**Mobile:**
```javascript
import * as FileSystem from 'expo-file-system';
const filePath = FileSystem.documentDirectory + 'myfile.json';
await FileSystem.writeAsStringAsync(filePath, JSON.stringify(value));
```

**Why different?** Web doesn't have `FileSystem`, mobile localStorage has size limits.

### SafeAreaView and Insets

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
            {/* Content */}
        </GestureHandlerRootView>
    );
}
```

**Rationale:** Android respects safe areas, but iOS doesn't need manual padding because SafeAreaProvider handles it.

### Header Height Adjustment

```javascript
const tabBarStyle = {
    backgroundColor: theme.surface,
    borderTopColor: theme.border,
    borderTopWidth: 1,
    paddingBottom: 0,
    height: 85,
    elevation: 0,           // Android shadow
    shadowColor: 'transparent', // iOS shadow
    shadowOpacity: 0,
    shadowOffset: { height: 0 },
    shadowRadius: 0,
};
```

Different shadow properties for iOS (shadow*) vs Android (elevation).

## Vibe Coding Insights

### 1. Context + Hooks > Props

Passing theme/language through props would require threading them through every component. Context + hooks eliminates prop drilling:

```javascript
// ❌ Without context
<App theme={theme} language={language}>
  <Navigation theme={theme} language={language}>
    <Screen theme={theme} language={language}>
      <Component theme={theme} language={language} />
    </Component>
  </Navigation>
</App>

// ✅ With context
<ThemeProvider>
  <LanguageProvider>
    <App>
      {/* Any component can useTheme() and useLanguage() */}
    </App>
  </LanguageProvider>
</ThemeProvider>
```

### 2. React Query Enables Offline-First

With proper React Query configuration, screens can display cached data immediately, then update in background when network is available. This is superior to showing loading spinners.

### 3. Platform Detection is Pragmatic

Rather than trying to create one solution for all platforms, detecting the platform and applying specific logic is often cleaner than abstraction layers.

## Summary

In Part 2, we explored:
- ✅ ThemeContext for theme switching with platform-specific persistence
- ✅ LanguageContext for smart language management
- ✅ RankingsProvider using React Query + local caching
- ✅ Reusable React Query hooks for different data types
- ✅ Platform-specific optimizations for iOS, Android, and Web

Key takeaway: **Separating concerns into context providers makes every component simpler**. By handling theme, language, and ranking management in dedicated contexts, screens like SearchScreen and DetailPage can focus purely on their business logic.

This is Vibe Coding in practice: using AI assistance to design clean, pragmatic solutions that work seamlessly across platforms while maintaining code clarity.
