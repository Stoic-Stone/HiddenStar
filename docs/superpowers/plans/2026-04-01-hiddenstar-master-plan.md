# HiddenStar V2 — Master Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a production-grade React Native mobile prototype for an AI-powered football talent detection platform that enables players to upload match videos, receive AI-driven performance analytics, and be discovered by scouts worldwide.

**Architecture:** Monorepo React Native (Expo managed workflow) app with Zustand state management, React Query for API layer, and Supabase as the BaaS (auth, database, storage). AI analysis is simulated in the prototype with mock data and realistic UI flows. The app supports two user roles (Player / Scout) with distinct navigation stacks.

**Tech Stack:**
- React Native (Expo SDK 52+)
- TypeScript
- Expo Router (file-based routing)
- Zustand (state management)
- React Query / TanStack Query (server state)
- Supabase (Auth, PostgreSQL, Storage)
- React Native Reanimated + Moti (animations)
- Expo AV / expo-image-picker (video)
- Victory Native (charts)

---

## 📐 Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    MOBILE APP (Expo)                     │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │  Auth     │  │  Player  │  │  Scout   │              │
│  │  Stack    │  │  Stack   │  │  Stack   │              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       │              │              │                    │
│  ┌────▼──────────────▼──────────────▼────┐              │
│  │        Zustand Store (Global)          │              │
│  │  ┌─────────┐ ┌──────────┐ ┌────────┐  │              │
│  │  │authStore│ │playerStore│ │uiStore │  │              │
│  │  └─────────┘ └──────────┘ └────────┘  │              │
│  └────────────────┬──────────────────────┘              │
│                   │                                      │
│  ┌────────────────▼──────────────────────┐              │
│  │      React Query (API Layer)           │              │
│  └────────────────┬──────────────────────┘              │
└───────────────────┼─────────────────────────────────────┘
                    │
        ┌───────────▼───────────┐
        │      SUPABASE         │
        │  ┌─────┐ ┌────────┐  │
        │  │Auth │ │Database│  │
        │  └─────┘ └────────┘  │
        │  ┌─────┐ ┌────────┐  │
        │  │Store│ │Edge Fn │  │
        │  └─────┘ └────────┘  │
        └───────────────────────┘
```

---

## 📁 Complete File Structure

```
hiddenstar-v2/
├── app/                          # Expo Router pages
│   ├── _layout.tsx               # Root layout (providers, fonts)
│   ├── index.tsx                 # Splash / entry redirect
│   ├── (auth)/                   # Auth group
│   │   ├── _layout.tsx
│   │   ├── login.tsx
│   │   ├── register.tsx
│   │   └── role-select.tsx
│   ├── (player)/                 # Player tab group
│   │   ├── _layout.tsx           # Tab navigator
│   │   ├── home.tsx
│   │   ├── upload.tsx
│   │   ├── analysis/
│   │   │   ├── [id].tsx          # Analysis result detail
│   │   │   └── index.tsx         # Analysis history list
│   │   └── profile.tsx
│   └── (scout)/                  # Scout tab group
│       ├── _layout.tsx           # Tab navigator
│       ├── dashboard.tsx
│       ├── players/
│       │   ├── index.tsx         # Player list + filters
│       │   └── [id].tsx          # Player detail
│       └── compare.tsx
├── components/                   # Reusable components
│   ├── ui/                       # Design system primitives
│   │   ├── Button.tsx
│   │   ├── Card.tsx
│   │   ├── Input.tsx
│   │   ├── Badge.tsx
│   │   ├── Avatar.tsx
│   │   ├── ProgressBar.tsx
│   │   ├── StatCard.tsx
│   │   ├── RadarChart.tsx
│   │   └── SkeletonLoader.tsx
│   ├── auth/
│   │   ├── LoginForm.tsx
│   │   ├── RegisterForm.tsx
│   │   └── SocialAuthButtons.tsx
│   ├── player/
│   │   ├── VideoUploader.tsx
│   │   ├── UploadProgress.tsx
│   │   ├── AnalysisCard.tsx
│   │   ├── PerformanceOverview.tsx
│   │   ├── MetricRow.tsx
│   │   └── MatchHistoryItem.tsx
│   ├── scout/
│   │   ├── PlayerRankingCard.tsx
│   │   ├── FilterSheet.tsx
│   │   ├── ComparisonTable.tsx
│   │   ├── ScoreDistribution.tsx
│   │   └── PlayerSearchBar.tsx
│   └── shared/
│       ├── Header.tsx
│       ├── EmptyState.tsx
│       ├── ErrorBoundary.tsx
│       └── LoadingOverlay.tsx
├── lib/                          # Core utilities
│   ├── supabase.ts               # Supabase client init
│   ├── queryClient.ts            # React Query config
│   └── constants.ts              # App constants, colors, sizes
├── stores/                       # Zustand stores
│   ├── authStore.ts
│   ├── playerStore.ts
│   ├── scoutStore.ts
│   └── uiStore.ts
├── hooks/                        # Custom hooks
│   ├── useAuth.ts
│   ├── useVideoUpload.ts
│   ├── useAnalysis.ts
│   ├── usePlayerProfile.ts
│   ├── useScoutDashboard.ts
│   └── useFilters.ts
├── services/                     # API service layer
│   ├── authService.ts
│   ├── videoService.ts
│   ├── analysisService.ts
│   ├── playerService.ts
│   └── scoutService.ts
├── types/                        # TypeScript types
│   ├── auth.ts
│   ├── player.ts
│   ├── analysis.ts
│   ├── video.ts
│   └── scout.ts
├── theme/                        # Design system tokens
│   ├── colors.ts
│   ├── typography.ts
│   ├── spacing.ts
│   └── shadows.ts
├── utils/                        # Pure utility functions
│   ├── formatters.ts
│   ├── validators.ts
│   ├── videoCompressor.ts
│   └── scoreCalculator.ts
├── assets/                       # Static assets
│   ├── fonts/
│   ├── images/
│   └── animations/               # Lottie files
├── __tests__/                    # Test files (mirrors src)
│   ├── components/
│   ├── hooks/
│   ├── stores/
│   ├── services/
│   └── utils/
├── app.json                      # Expo config
├── tsconfig.json
├── babel.config.js
├── package.json
└── eas.json                      # EAS Build config
```

---

## 🗺️ Sub-Plan Registry

This master plan is decomposed into **6 independent sub-plans**. Each produces working, testable software. Execute them in dependency order.

| # | Sub-Plan | File | Depends On | Phase |
|---|----------|------|------------|-------|
| 1 | **Project Scaffolding & Design System** | `plan-01-scaffolding.md` | — | Week 1 |
| 2 | **Authentication & Onboarding** | `plan-02-authentication.md` | Plan 1 | Week 1–2 |
| 3 | **Video Upload System** | `plan-03-video-upload.md` | Plan 1, 2 | Week 2–3 |
| 4 | **Player Profile & Data Layer** | `plan-04-player-profile.md` | Plan 1, 2 | Week 2–3 |
| 5 | **Scout Dashboard & Discovery** | `plan-05-scout-dashboard.md` | Plan 1, 2, 4 | Week 3–4 |
| 6 | **AI Analysis Integration** | `plan-06-ai-analysis.md` | Plan 1, 2, 3, 4 | Week 4–5 |

### Dependency Graph

```
Plan 1 (Scaffolding) ──► Plan 2 (Auth) ──► Plan 3 (Video Upload) ──► Plan 6 (AI Analysis)
         │                    │                                            ▲
         │                    ▼                                            │
         └──────────────► Plan 4 (Player Profile) ────► Plan 5 (Scout) ───┘
```

---

## 🗃️ Database Schema (Supabase PostgreSQL)

```sql
-- Users extended profile (Supabase Auth handles core auth)
CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  full_name TEXT NOT NULL,
  role TEXT NOT NULL CHECK (role IN ('player', 'scout')),
  avatar_url TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Player-specific profile data
CREATE TABLE player_profiles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  age INTEGER CHECK (age >= 10 AND age <= 25),
  position TEXT NOT NULL,
  height_cm NUMERIC,
  weight_kg NUMERIC,
  preferred_foot TEXT CHECK (preferred_foot IN ('left', 'right', 'both')),
  club_name TEXT,
  country TEXT,
  bio TEXT,
  UNIQUE(user_id)
);

-- Uploaded videos
CREATE TABLE videos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  storage_path TEXT NOT NULL,
  thumbnail_url TEXT,
  duration_seconds INTEGER,
  file_size_bytes BIGINT,
  status TEXT DEFAULT 'uploaded' CHECK (status IN ('uploaded', 'processing', 'completed', 'failed')),
  uploaded_at TIMESTAMPTZ DEFAULT NOW()
);

-- AI analysis results
CREATE TABLE analyses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
  distance_covered NUMERIC,
  avg_speed NUMERIC,
  sprint_count INTEGER,
  ball_touches INTEGER,
  pass_accuracy NUMERIC,
  shot_attempts INTEGER,
  movement_score NUMERIC,
  technical_score NUMERIC,
  decision_score NUMERIC,
  activity_score NUMERIC,
  composite_score NUMERIC,
  processing_started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for scout queries
CREATE INDEX idx_analyses_composite ON analyses(composite_score DESC);
CREATE INDEX idx_player_profiles_position ON player_profiles(position);
CREATE INDEX idx_analyses_user ON analyses(user_id);
CREATE INDEX idx_videos_user ON videos(user_id);
```

---

## 🎨 Design System Tokens (Preview)

```typescript
// theme/colors.ts
export const colors = {
  // Primary — Electric green (football pitch energy)
  primary: {
    50:  '#E8FFF0',
    100: '#B8FFCF',
    200: '#7DFFAA',
    300: '#3DFF82',
    400: '#00FF5A',  // Main accent
    500: '#00E050',
    600: '#00B840',
    700: '#008F30',
    800: '#006620',
    900: '#003D10',
  },
  // Dark — Deep space (premium feel)
  dark: {
    50:  '#E6E8EB',
    100: '#BFC4CB',
    200: '#959DA8',
    300: '#6B7685',
    400: '#4B586A',
    500: '#2B3A50',
    600: '#1E2D42',
    700: '#152036',  // Card background
    800: '#0D1526',  // Screen background
    900: '#060A14',  // Deepest
  },
  // Accent — Amber (highlights & warnings)
  accent: {
    400: '#FFB020',
    500: '#FF9500',
  },
  // Semantic
  success: '#00C853',
  error:   '#FF3B30',
  warning: '#FF9500',
  info:    '#007AFF',
  // Neutrals
  white:   '#FFFFFF',
  text: {
    primary:   '#FFFFFF',
    secondary: '#A0AEC0',
    muted:     '#6B7685',
  },
};
```

---

## ✅ Quality Gates

Before merging each sub-plan:

1. **All tests pass** — `npm test` green
2. **TypeScript strict** — zero `any` types, zero `ts-ignore`
3. **No console.log** — use proper logging or remove
4. **Accessibility** — all touchable elements have `accessibilityLabel`
5. **Performance** — no unnecessary re-renders (React DevTools profiler)
6. **EAS Build** — `eas build --platform ios --profile preview` succeeds

---

## 🚀 Delivery Milestones

| Milestone | Deliverable | Week |
|-----------|------------|------|
| M1 | App boots with design system, splash screen, navigation shell | 1 |
| M2 | Full auth flow (email + social) with role routing | 2 |
| M3 | Video upload with compression, progress, and storage | 3 |
| M4 | Player profile with match history and stats display | 3 |
| M5 | Scout dashboard with rankings, filters, and comparison | 4 |
| M6 | AI analysis integration with simulated pipeline and results UI | 5 |
| M7 | Polish, animations, error handling, and prototype demo build | 5 |

---

> **Next Step:** Read each sub-plan document in order. Start with `plan-01-scaffolding.md`.
