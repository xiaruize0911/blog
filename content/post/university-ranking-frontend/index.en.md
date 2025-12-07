---
title: "Building University Ranking Frontend with Vibe Coding: Architecture & Design Strategy"
date: 2025-12-07
draft: false
tags: ["React Native", "Vibe Coding", "Mobile Development", "Frontend", "Expo"]
categories: ["Building an App using vibe coding"]
description: "Explore how we built the University Ranking Frontend as a React Native mobile application using Vibe Coding—AI-assisted development that emphasizes pragmatic choices and rapid iteration."
---

## Introduction

The University Ranking Frontend is a sophisticated React Native application that demonstrates how to build production-grade mobile experiences using **Vibe Coding**—a development philosophy where AI assistance guides architectural decisions, simplifies state management, and accelerates feature implementation.

Unlike building a backend service (which requires careful database design and algorithm optimization), frontend development is often about creating intuitive user experiences and managing complex application state. In this series, we'll explore how Vibe Coding transforms frontend development from a labor-intensive process into an iterative, AI-assisted collaboration that produces clean, maintainable code.

## What is Vibe Coding?

Vibe Coding means **building software in collaboration with AI**, leveraging machine learning models to:

- **Generate architecture patterns** that match the problem space
- **Suggest state management solutions** appropriate for your data flow
- **Accelerate component development** through intelligent code generation
- **Validate design patterns** and identify potential improvements before implementation
- **Iterate rapidly** on features with AI-generated boilerplate and scaffolding

For the University Ranking Frontend, Vibe Coding helped us:

1. **Choose React Native + Expo** over web or native development paths, based on requirements for cross-platform support (iOS, Android, Web)
2. **Design the context API system** for theme, language, and rankings management without over-engineering
3. **Structure API integration** using React Query for efficient data fetching and caching
4. **Build UI components** that adapt seamlessly to dark mode and language switching
5. **Implement search with filters** that feel responsive and native-like

## Project Overview

The University Ranking Frontend is a mobile-first application that allows users to:

- **Search universities** by name, country, or city with real-time filtering
- **View detailed rankings** across multiple ranking systems (QS, US News, etc.)
- **Toggle themes** (light/dark mode) with persistent preferences
- **Switch languages** (English/Chinese) with full UI localization
- **Save favorites** for quick access to frequently viewed universities
- **Compare rankings** across different metrics and sources

### Technology Stack

**Core Framework:**
- **React Native**: Cross-platform mobile development
- **Expo**: Development platform and managed build system for React Native
- **@react-navigation**: Navigation library with bottom tab and stack navigators

**State Management:**
- **React Context API**: Theme, Language, and Rankings state
- **@tanstack/react-query**: Server state management with caching and background syncing
- **AsyncStorage / Expo FileSystem**: Local persistence for preferences and cache

**UI & Styling:**
- **React Native built-in components**: View, Text, FlatList, ScrollView, etc.
- **@gorhom/bottom-sheet**: Native bottom sheet modal for filters
- **@expo/vector-icons**: Icon library (Ionicons)

**Localization & API:**
- **i18n-js**: Internationalization for English/Chinese support
- **axios**: HTTP client for API requests
- **Bing Translator API**: Dynamic content translation (e.g., university descriptions)

### Application Architecture

```
┌─────────────────────────────────────────┐
│         App.js (Root Navigator)         │
│  - Query Client Configuration           │
│  - Context Providers Setup              │
└─────────────────────────────────────────┘
                    ↓
    ┌───────────────────────────────┐
    │   Bottom Tab Navigation       │
    │  ┌─────────────────────────┐  │
    │  │  Search Stack           │  │
    │  │  - SearchScreen         │  │
    │  │  - DetailPage           │  │
    │  └─────────────────────────┘  │
    │  ┌─────────────────────────┐  │
    │  │  Rankings Stack         │  │
    │  │  - SubjectRankings      │  │
    │  │  - RankingDetailPage    │  │
    │  └─────────────────────────┘  │
    │  ┌─────────────────────────┐  │
    │  │  Other Stacks           │  │
    │  │  - Favorites            │  │
    │  │  - Me (Profile)         │  │
    │  └─────────────────────────┘  │
    └───────────────────────────────┘
                    ↓
    ┌───────────────────────────────┐
    │   Context Providers           │
    │  ┌─────────────────────────┐  │
    │  │  ThemeProvider          │  │
    │  │  - Dark/Light themes    │  │
    │  │  - Platform-specific    │  │
    │  └─────────────────────────┘  │
    │  ┌─────────────────────────┐  │
    │  │  LanguageProvider       │  │
    │  │  - en/zh support        │  │
    │  │  - Device locale detect │  │
    │  └─────────────────────────┘  │
    │  ┌─────────────────────────┐  │
    │  │  RankingsProvider       │  │
    │  │  - Subject rankings     │  │
    │  │  - Cached/synced        │  │
    │  └─────────────────────────┘  │
    └───────────────────────────────┘
                    ↓
    ┌───────────────────────────────┐
    │   API Integration Layer       │
    │  - React Query hooks          │
    │  - Raw axios functions        │
    │  - Local file caching         │
    └───────────────────────────────┘
```

## Building Strategy with Vibe Coding

### Phase 1: Foundation (Navigation & Context Setup)

**Challenge**: How to structure a multi-screen app where users can search, view details, compare rankings, and manage preferences?

**Vibe Coding Approach**: Rather than over-engineering a complex state management solution, we used:
- **Bottom Tab Navigation** for major sections (Search, Rankings, Favorites, Me)
- **Stack navigators** within each tab for hierarchical screens
- **React Context API** for global state that's accessed everywhere (Theme, Language, Rankings)

This pragmatic choice avoids Redux/Mobx complexity while maintaining clean separation of concerns.

### Phase 2: State Management (Context Patterns)

**Challenge**: How to manage theme switching, language preferences, and ranking options across the entire app without prop drilling?

**Vibe Coding Approach**: Three separate context providers, each handling a specific concern:

1. **ThemeContext**: Manages dark/light mode with platform-specific styling (Android/iOS/Web)
2. **LanguageContext**: Handles language switching with device locale detection
3. **RankingsProvider**: Uses React Query for ranking options with local caching (FileSystem on native, localStorage on web)

Each context is self-contained and can be tested independently. New features can hook into existing contexts without disruption.

### Phase 3: API & Data Fetching

**Challenge**: How to fetch university data efficiently, cache results, and keep the UI responsive?

**Vibe Coding Approach**: We layered two API patterns:

1. **Raw API functions** (`lib/api.js`): Basic axios calls for direct data fetching
2. **React Query hooks**: Wrapping raw functions with caching, background sync, and automatic retry logic

This two-layer approach means:
- Simple components can use raw API calls for one-off requests
- Complex screens use React Query for sophisticated caching strategies
- We can refactor without touching screen code

### Phase 4: UI Implementation

**Challenge**: How to create a responsive UI that works on phones, tablets, and web while supporting dark mode and multiple languages?

**Vibe Coding Approach**: Component composition with context-aware styling:
- **Theme-aware styling**: Every component reads colors from ThemeContext
- **Language-aware text**: All UI strings use i18n for instant language switching
- **Platform-specific adjustments**: Layout tweaks for iOS, Android, and Web
- **Bottom sheet modals**: For filters and selections, providing native-like UX

## Key Insights

### 1. React Context is Sufficient

Many developers jump to Redux or Mobx for state management. For the University Ranking Frontend, React Context + hooks proved sufficient for:
- Global theme/language state
- Navigation-aware state
- Lazy-loaded ranking options

We avoided over-engineering—a Vibe Coding principle.

### 2. React Query Transforms Data Management

Instead of manually managing API request states (loading, error, success), React Query handles:
- Request deduplication
- Automatic retries
- Stale data invalidation
- Background synchronization

This dramatically reduced boilerplate and improved reliability.

### 3. Expo Simplifies Cross-Platform Development

Rather than maintaining separate iOS/Android native projects, Expo provides:
- Single JavaScript codebase
- Managed builds (no Xcode/Android Studio needed)
- Over-the-air updates capability
- Web support through React Native Web

This aligns perfectly with Vibe Coding—letting AI handle platform differences while developers focus on logic.

### 4. Localization Must Be Planned Early

Supporting multiple languages isn't an afterthought. The University Ranking Frontend is bilingual (English/Chinese) from day one:
- i18n-js handles string translations
- UniversityTranslationsContext caches university name translations
- API integration includes translation endpoints (Bing Translator)
- Device locale is auto-detected for seamless UX

## Architecture Highlights

### Bottom Tab Navigation

The main navigation structure uses `@react-navigation/bottom-tabs`:

```javascript
<NavigationContainer>
  <Tab.Navigator
    screenOptions={({ route }) => ({
      tabBarIcon: ({ focused, color, size }) => {
        // Dynamic icon selection based on route
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

Each tab has its own Stack Navigator for detailed views.

### Context Provider Composition

In `App.js`, contexts are nested to provide data to all screens:

```javascript
<QueryClientProvider client={queryClient}>
  <ThemeProvider>
    <LanguageProvider>
      <UniversityTranslationsProvider>
        <RankingsProvider>
          <NavigationContainer>
            {/* Navigation structure */}
          </NavigationContainer>
        </RankingsProvider>
      </UniversityTranslationsProvider>
    </LanguageProvider>
  </ThemeProvider>
</QueryClientProvider>
```

This composition order matters: ThemeProvider and LanguageProvider should wrap everything since they provide foundational styling.

## What's Next

In {{< ref "university-ranking-frontend-part-1" >}}, we'll dive deep into:
- How the navigation orchestration works in `App.js`
- Building the SearchScreen with real-time filtering and bottom sheet modals
- Implementing the DetailPage with lazy image loading and dynamic translations
- Navigation patterns that keep the app responsive

In {{< ref "university-ranking-frontend-part-2" >}}, we'll explore:
- Context implementation details (ThemeContext, LanguageContext, RankingsProvider)
- React Query patterns for efficient data fetching
- Component architecture for theme-aware UI
- Handling platform-specific behaviors (iOS/Android/Web)

---

## Summary

The University Ranking Frontend demonstrates that building sophisticated mobile applications doesn't require complex frameworks. With Vibe Coding, we leveraged AI to:

1. Choose the right technology stack (React Native + Expo)
2. Design scalable state management (Context + React Query)
3. Build responsive, bilingual UI with minimal boilerplate
4. Create a production-ready app that works on iOS, Android, and Web

The key insight: **Good architecture is pragmatic, not theoretical**. Vibe Coding helps us make these pragmatic choices quickly and confidently.

In the next parts of this series, we'll dissect the code and show exactly how each piece fits together.
