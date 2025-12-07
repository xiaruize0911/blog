---
title: "University Ranking Frontend Deep Dive (Part 1): Navigation Architecture & Search Implementation"
date: 2025-12-07
draft: false
tags: ["React Native", "Navigation", "Search", "Vibe Coding", "Implementation"]
categories: ["Building an App using vibe coding"]
description: "Part 1 of our deep dive into the University Ranking Frontend. We explore the navigation orchestration in App.js, implement the SearchScreen with real-time filtering and bottom sheet modals, and build the DetailPage for university profiles."
---

This is Part 1 of our University Ranking Frontend code deep dive. If you haven't read the {{< ref "university-ranking-frontend" >}} introduction yet, I recommend starting there.

## Overview

In Part 1, we'll explore:
1. **Navigation Orchestration** - How `App.js` orchestrates bottom tab and stack navigators
2. **SearchScreen Implementation** - Building the main search interface with real-time filtering
3. **DetailPage Construction** - Displaying comprehensive university profiles with dynamic content
4. **API Integration** - Fetching data with axios and handling async operations

## Architecture Lesson: Why We Use This Navigation Pattern

Many developers overthink navigation. The University Ranking Frontend uses a simple but powerful pattern:

```
Bottom Tabs
├── Home (Search)
│   └── Stack Navigator
│       ├── SearchScreen
│       ├── DetailPage
│       └── UniversitySourceRankingsPage
├── Subject Rankings
│   └── Stack Navigator
│       ├── SubjectRankingsPage
│       ├── RankingDetailPage
│       └── DetailPage
├── Favorites
│   └── Stack Navigator
│       └── FavoritesScreen
└── Me (Profile)
    └── Stack Navigator
        └── MeScreen
```

**Why this works:**
- **Bottom tabs** provide persistent navigation between major app sections
- **Stack navigators** within each tab enable push/pop transitions for detail views
- **Shared screens** (DetailPage appears in multiple stacks) reuse components across contexts

This is **Vibe Coding in action**: instead of inventing a custom navigation system, we use proven React Navigation patterns that AI can quickly suggest and improve.

## App.js: The Navigation Orchestrator

Let's examine the root App.js file, which sets up the entire navigation hierarchy:

```javascript
import React, { useState, useEffect } from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { BottomSheetModalProvider } from '@gorhom/bottom-sheet';
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

// Initialize React Query Client
const queryClient = new QueryClient({
	defaultOptions: {
		queries: {
			staleTime: 5 * 60 * 1000, // 5 minutes
			cacheTime: 10 * 60 * 1000, // 10 minutes
			retry: 2,
			refetchOnWindowFocus: false,
		},
	},
});

const Stack = createNativeStackNavigator();
const Tab = createBottomTabNavigator();
```

Key configuration decisions:

1. **Stale Time (5 minutes)**: After 5 minutes, cached data is considered stale and React Query will refresh in the background
2. **Cache Time (10 minutes)**: Keep cached data for 10 minutes before garbage collection
3. **Retry (2)**: Automatically retry failed requests twice before showing an error
4. **refetchOnWindowFocus (false)**: Don't refetch when the app regains focus (for better UX)

This configuration is pragmatic: it balances responsiveness with network efficiency.

### Home Stack (SearchScreen)

The Home stack is the primary navigation entry point:

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

**Key Pattern**: The header styling is theme-aware. Whenever the user toggles dark mode, these colors update in real-time because we read from the `useTheme()` hook.

### Context Provider Composition

App.js wraps the entire navigation tree with context providers:

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

**Order matters:**
1. **QueryClientProvider** (outermost) - Manages all server state
2. **ThemeProvider** - Provides color scheme to all components
3. **LanguageProvider** - Provides language and translation functions
4. **UniversityTranslationsProvider** - Provides cached university name translations
5. **RankingsProvider** - Provides ranking options for filters

Each provider is independent and can be tested in isolation.

## SearchScreen: Real-Time University Search

The SearchScreen is the most complex component in the app. It implements:
- Real-time search with debouncing
- Filter selection with bottom sheets
- Animated loading spinner
- Responsive layout with platform-specific adjustments

### Data Fetching with Effects

```javascript
export default function SearchScreen() {
    const [query, setQuery] = useState('');
    const [country, setCountry] = useState(null);
    const [sortCredit, setSortCredit] = useState('QS_World_University_Rankings');
    const [results, setResults] = useState([]);
    const [isLoading, setIsLoading] = useState(false);
    
    // Fetch universities when filters change
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
                
                if (!isMounted) return; // Don't update if unmounted
                
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
            isMounted = false; // Prevent updates on unmounted component
        };
    }, [query, country, sortCredit]); // Re-run when filters change
}
```

**Vibe Coding Pattern**: The `isMounted` flag prevents memory leaks and race conditions. AI suggested this pattern because:
- Navigation can unmount a screen before async operations complete
- Without `isMounted`, state updates on unmounted components cause warnings
- This pattern is idiomatic for React hooks

### Bottom Sheet Modals for Filters

The SearchScreen uses bottom sheet modals for country and ranking selection:

```javascript
import { BottomSheetModal, BottomSheetScrollView, BottomSheetBackdrop } from '@gorhom/bottom-sheet';

const countrySheetRef = useRef(null);
const rankingSheetRef = useRef(null);
const snapPoints = useMemo(() => ['60%', '75%', '90%'], []);

// Open country filter
const handleCountryPress = useCallback(() => {
    setCountrySearchQuery('');
    rankingSheetRef.current?.dismiss(); // Close other sheet
    countrySheetRef.current?.present();
}, []);

// Filter countries as user types
const filteredCountryItems = useMemo(() => {
    if (!countrySearchQuery) return countryItems;
    return countryItems.filter(item =>
        item.label.toLowerCase().includes(countrySearchQuery.toLowerCase())
    );
}, [countryItems, countrySearchQuery]);
```

**Key insight**: Bottom sheets provide a native-like UX where filters are accessible but don't hide the main content. The `snapPoints` allow users to drag the sheet to different heights.

### Search Results Display

The search results are displayed in a FlatList for efficient rendering of large lists:

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

**Pattern**: Using `normalized_name` as both the list key and the identifier passed to DetailPage ensures consistency across the app.

## DetailPage: Comprehensive University Profiles

The DetailPage displays:
- University logo and basic information
- Rankings across multiple sources
- University description (dynamically translated if needed)
- Favorite/unfavorite functionality
- Links to detailed ranking breakdowns

### Async Data Loading

```javascript
export default function GetUniversityDetail(props) {
    const normalized_name = props.route.params.normalized_name;
    const [university, setUniversity] = useState(null);
    const [loading, setLoading] = useState(true);
    const [translatedBlurb, setTranslatedBlurb] = useState('');
    const [translationLoading, setTranslationLoading] = useState(false);
    
    const { isChinese } = useLanguage();

    // Fetch university details
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

    // Translate description when language changes
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

**Key Pattern**: Translations are handled as a side effect when the language context changes. This keeps translation logic separate from component rendering.

### Favorites Management with AsyncStorage

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
            // Remove from favorites
            favorites = favorites.filter(fav =>
                fav.normalized_name !== university.normalized_name &&
                fav.id !== university.id
            );
            setIsFavorite(false);
        } else {
            // Add to favorites
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

**Vibe Coding insight**: Rather than syncing favorites to the backend, we store them locally with AsyncStorage. This provides:
- Instant feedback (no network delay)
- Works offline
- Reduces server load

The `favoriteLoading` state ensures the UI shows loading state during the toggle operation.

## API Integration: The lib/api.js Layer

The API layer separates API calls from components:

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

// Export functions for direct use
export const searchUniversities = rawApi.searchUniversities;
export const getUniversityDetails = rawApi.getUniversityDetails;
export const translateText = rawApi.translateText;
```

**Benefits of this pattern:**
- **Testability**: Easy to mock `rawApi` for unit tests
- **Reusability**: API functions can be used directly or wrapped with React Query
- **Maintainability**: All API calls in one place

## Design Insights from Vibe Coding

### 1. Real-Time Search is Easier Than Debouncing

Initially, debouncing the search seemed necessary to reduce API calls. However, the backend filter endpoint is fast, and the network is reliable. So we skip debouncing and let the user see instant results. **This is Vibe Coding**: pragmatic choices over theoretical optimization.

### 2. Bottom Sheets > Modal Dialogs

Native-like bottom sheets provide better UX than centered modals because:
- Users can see content behind the sheet
- Dragging to dismiss feels natural
- Screen real estate is used efficiently

AI suggested `@gorhom/bottom-sheet` because it provides smooth animations and gesture handling.

### 3. Context + AsyncStorage > Redux for Preferences

Managing favorites locally with AsyncStorage and React Context is simpler than:
- Redux + Redux Persist
- Syncing to backend
- Complex state management

Vibe Coding helped us recognize that not everything needs to be in global state.

## Summary

In Part 1, we explored:
- ✅ How navigation is orchestrated in App.js with bottom tabs and stack navigators
- ✅ Building SearchScreen with real-time filtering and bottom sheet modals
- ✅ Implementing DetailPage with async data loading and favorites management
- ✅ Structuring API calls for reusability and testability

In {{< ref "university-ranking-frontend-part-2" >}}, we'll deep dive into:
- Context implementations (ThemeContext, LanguageContext, RankingsProvider)
- React Query patterns for sophisticated data management
- Component styling patterns for theme awareness
- Platform-specific considerations

The key takeaway: **Good navigation architecture enables clean component design**. By handling navigation concerns in App.js, individual screens can focus on their specific responsibilities—searching, displaying, and managing data.
