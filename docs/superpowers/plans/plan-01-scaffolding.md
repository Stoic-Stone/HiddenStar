# Plan 01: Project Scaffolding & Design System

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Bootstrap the Expo React Native project with TypeScript, install all dependencies, create the design system tokens, build reusable UI primitives, and wire up the navigation shell with role-based routing.

**Architecture:** Expo managed workflow with file-based routing (expo-router). Design tokens defined in `theme/` drive all visual decisions. UI primitives in `components/ui/` are the building blocks for every screen.

**Tech Stack:** Expo SDK 52+, TypeScript, Expo Router, React Native Reanimated, Moti, expo-font, @expo/vector-icons

---

### Task 1: Initialize Expo Project

**Files:**
- Create: `hiddenstar-v2/` (project root)

- [ ] **Step 1: Create Expo project**

```bash
npx -y create-expo-app@latest ./hiddenstar-v2 --template tabs
```

- [ ] **Step 2: Navigate and install core dependencies**

```bash
cd hiddenstar-v2
npx expo install expo-router expo-font expo-splash-screen expo-status-bar react-native-reanimated react-native-gesture-handler react-native-safe-area-context react-native-screens expo-linking expo-constants
npm install zustand @tanstack/react-query axios moti @supabase/supabase-js react-native-url-polyfill @react-native-async-storage/async-storage
npm install -D @types/react @testing-library/react-native jest
```

- [ ] **Step 3: Verify project runs**

```bash
npx expo start
```
Expected: Dev server launches, app renders default tab screen on simulator.

- [ ] **Step 4: Commit**

```bash
git init && git add -A && git commit -m "feat: initialize expo project with core dependencies"
```

---

### Task 2: Design System — Color Tokens

**Files:**
- Create: `theme/colors.ts`
- Test: `__tests__/theme/colors.test.ts`

- [ ] **Step 1: Write the test**

```typescript
// __tests__/theme/colors.test.ts
import { colors } from '../theme/colors';

describe('Color tokens', () => {
  it('has primary palette with 400 as main accent', () => {
    expect(colors.primary[400]).toBe('#00FF5A');
  });
  it('has dark palette with 800 as screen background', () => {
    expect(colors.dark[800]).toBe('#0D1526');
  });
  it('has semantic colors', () => {
    expect(colors.success).toBeDefined();
    expect(colors.error).toBeDefined();
  });
  it('has text hierarchy', () => {
    expect(colors.text.primary).toBe('#FFFFFF');
    expect(colors.text.secondary).toBeDefined();
    expect(colors.text.muted).toBeDefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx jest __tests__/theme/colors.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement colors**

```typescript
// theme/colors.ts
export const colors = {
  primary: {
    50: '#E8FFF0', 100: '#B8FFCF', 200: '#7DFFAA', 300: '#3DFF82',
    400: '#00FF5A', 500: '#00E050', 600: '#00B840', 700: '#008F30',
    800: '#006620', 900: '#003D10',
  },
  dark: {
    50: '#E6E8EB', 100: '#BFC4CB', 200: '#959DA8', 300: '#6B7685',
    400: '#4B586A', 500: '#2B3A50', 600: '#1E2D42', 700: '#152036',
    800: '#0D1526', 900: '#060A14',
  },
  accent: { 400: '#FFB020', 500: '#FF9500' },
  success: '#00C853',
  error: '#FF3B30',
  warning: '#FF9500',
  info: '#007AFF',
  white: '#FFFFFF',
  text: { primary: '#FFFFFF', secondary: '#A0AEC0', muted: '#6B7685' },
} as const;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx jest __tests__/theme/colors.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add theme/colors.ts __tests__/theme/colors.test.ts
git commit -m "feat: add color design tokens"
```

---

### Task 3: Design System — Typography & Spacing

**Files:**
- Create: `theme/typography.ts`
- Create: `theme/spacing.ts`
- Create: `theme/shadows.ts`

- [ ] **Step 1: Create typography tokens**

```typescript
// theme/typography.ts
export const typography = {
  fonts: {
    heading: 'Outfit_700Bold',
    body: 'Inter_400Regular',
    bodyMedium: 'Inter_500Medium',
    bodySemiBold: 'Inter_600SemiBold',
    mono: 'JetBrainsMono_400Regular',
  },
  sizes: {
    xs: 11, sm: 13, base: 15, md: 17, lg: 20, xl: 24, '2xl': 30, '3xl': 36, '4xl': 48,
  },
  lineHeights: {
    tight: 1.2, normal: 1.5, relaxed: 1.75,
  },
} as const;
```

- [ ] **Step 2: Create spacing tokens**

```typescript
// theme/spacing.ts
export const spacing = {
  xs: 4, sm: 8, md: 12, base: 16, lg: 20, xl: 24, '2xl': 32, '3xl': 40, '4xl': 48, '5xl': 64,
} as const;

export const borderRadius = {
  sm: 6, md: 10, lg: 14, xl: 20, full: 9999,
} as const;
```

- [ ] **Step 3: Create shadow tokens**

```typescript
// theme/shadows.ts
import { Platform } from 'react-native';

export const shadows = {
  sm: Platform.select({
    ios: { shadowColor: '#000', shadowOffset: { width: 0, height: 1 }, shadowOpacity: 0.15, shadowRadius: 3 },
    android: { elevation: 2 },
  }),
  md: Platform.select({
    ios: { shadowColor: '#000', shadowOffset: { width: 0, height: 4 }, shadowOpacity: 0.2, shadowRadius: 8 },
    android: { elevation: 5 },
  }),
  lg: Platform.select({
    ios: { shadowColor: '#000', shadowOffset: { width: 0, height: 8 }, shadowOpacity: 0.25, shadowRadius: 16 },
    android: { elevation: 10 },
  }),
  glow: (color: string) => Platform.select({
    ios: { shadowColor: color, shadowOffset: { width: 0, height: 0 }, shadowOpacity: 0.5, shadowRadius: 12 },
    android: { elevation: 8 },
  }),
} as const;
```

- [ ] **Step 4: Commit**

```bash
git add theme/
git commit -m "feat: add typography, spacing, and shadow tokens"
```

---

### Task 4: Install & Load Custom Fonts

**Files:**
- Modify: `app/_layout.tsx`

- [ ] **Step 1: Install font packages**

```bash
npx expo install @expo-google-fonts/outfit @expo-google-fonts/inter expo-font expo-splash-screen
```

- [ ] **Step 2: Update root layout to load fonts**

```tsx
// app/_layout.tsx
import { useEffect } from 'react';
import { Stack } from 'expo-router';
import * as SplashScreen from 'expo-splash-screen';
import { useFonts, Outfit_700Bold, Outfit_600SemiBold } from '@expo-google-fonts/outfit';
import { Inter_400Regular, Inter_500Medium, Inter_600SemiBold as InterSemiBold } from '@expo-google-fonts/inter';
import { StatusBar } from 'expo-status-bar';
import { View } from 'react-native';
import { colors } from '../theme/colors';

SplashScreen.preventAutoHideAsync();

export default function RootLayout() {
  const [fontsLoaded] = useFonts({
    Outfit_700Bold,
    Outfit_600SemiBold,
    Inter_400Regular,
    Inter_500Medium,
    Inter_600SemiBold: InterSemiBold,
  });

  useEffect(() => {
    if (fontsLoaded) SplashScreen.hideAsync();
  }, [fontsLoaded]);

  if (!fontsLoaded) return null;

  return (
    <View style={{ flex: 1, backgroundColor: colors.dark[800] }}>
      <StatusBar style="light" />
      <Stack screenOptions={{ headerShown: false, contentStyle: { backgroundColor: colors.dark[800] } }} />
    </View>
  );
}
```

- [ ] **Step 3: Verify fonts render** — run app and confirm no font warnings in console.

- [ ] **Step 4: Commit**

```bash
git add app/_layout.tsx
git commit -m "feat: load custom fonts (Outfit + Inter)"
```

---

### Task 5: Build Button Component

**Files:**
- Create: `components/ui/Button.tsx`
- Test: `__tests__/components/ui/Button.test.tsx`

- [ ] **Step 1: Write the test**

```tsx
// __tests__/components/ui/Button.test.tsx
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import { Button } from '../../components/ui/Button';

describe('Button', () => {
  it('renders label text', () => {
    const { getByText } = render(<Button label="Upload" onPress={() => {}} />);
    expect(getByText('Upload')).toBeTruthy();
  });
  it('calls onPress when tapped', () => {
    const onPress = jest.fn();
    const { getByText } = render(<Button label="Tap Me" onPress={onPress} />);
    fireEvent.press(getByText('Tap Me'));
    expect(onPress).toHaveBeenCalledTimes(1);
  });
  it('is disabled when loading', () => {
    const onPress = jest.fn();
    const { getByText } = render(<Button label="Save" onPress={onPress} loading />);
    fireEvent.press(getByText('Save'));
    expect(onPress).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Implement Button**

```tsx
// components/ui/Button.tsx
import React from 'react';
import { TouchableOpacity, Text, ActivityIndicator, StyleSheet, ViewStyle, TextStyle } from 'react-native';
import { colors } from '../../theme/colors';
import { typography } from '../../theme/typography';
import { borderRadius, spacing } from '../../theme/spacing';

type Variant = 'primary' | 'secondary' | 'outline' | 'ghost';

interface ButtonProps {
  label: string;
  onPress: () => void;
  variant?: Variant;
  loading?: boolean;
  disabled?: boolean;
  fullWidth?: boolean;
  accessibilityLabel?: string;
}

export function Button({ label, onPress, variant = 'primary', loading = false, disabled = false, fullWidth = false, accessibilityLabel }: ButtonProps) {
  const isDisabled = disabled || loading;

  const containerStyle: ViewStyle[] = [
    styles.base,
    styles[variant],
    fullWidth && styles.fullWidth,
    isDisabled && styles.disabled,
  ].filter(Boolean) as ViewStyle[];

  const textColor = variant === 'primary' ? colors.dark[900] : variant === 'outline' || variant === 'ghost' ? colors.primary[400] : colors.white;

  return (
    <TouchableOpacity
      style={containerStyle}
      onPress={onPress}
      disabled={isDisabled}
      activeOpacity={0.8}
      accessibilityLabel={accessibilityLabel || label}
      accessibilityRole="button"
    >
      {loading ? (
        <ActivityIndicator color={textColor} size="small" />
      ) : (
        <Text style={[styles.label, { color: textColor, fontFamily: typography.fonts.bodySemiBold }]}>{label}</Text>
      )}
    </TouchableOpacity>
  );
}

const styles = StyleSheet.create({
  base: { height: 52, borderRadius: borderRadius.lg, alignItems: 'center', justifyContent: 'center', paddingHorizontal: spacing.xl, flexDirection: 'row' },
  primary: { backgroundColor: colors.primary[400] },
  secondary: { backgroundColor: colors.dark[600] },
  outline: { backgroundColor: 'transparent', borderWidth: 1.5, borderColor: colors.primary[400] },
  ghost: { backgroundColor: 'transparent' },
  fullWidth: { width: '100%' },
  disabled: { opacity: 0.5 },
  label: { fontSize: typography.sizes.md },
});
```

- [ ] **Step 4: Run test — expect PASS**
- [ ] **Step 5: Commit**

```bash
git add components/ui/Button.tsx __tests__/components/ui/Button.test.tsx
git commit -m "feat: add Button UI component"
```

---

### Task 6: Build Card, Input, Badge, Avatar Components

**Files:**
- Create: `components/ui/Card.tsx`
- Create: `components/ui/Input.tsx`
- Create: `components/ui/Badge.tsx`
- Create: `components/ui/Avatar.tsx`

- [ ] **Step 1: Create Card**

```tsx
// components/ui/Card.tsx
import React from 'react';
import { View, StyleSheet, ViewStyle } from 'react-native';
import { colors } from '../../theme/colors';
import { borderRadius, spacing } from '../../theme/spacing';
import { shadows } from '../../theme/shadows';

interface CardProps { children: React.ReactNode; style?: ViewStyle; variant?: 'default' | 'elevated' }

export function Card({ children, style, variant = 'default' }: CardProps) {
  return (
    <View style={[styles.card, variant === 'elevated' && shadows.md, style]}>
      {children}
    </View>
  );
}

const styles = StyleSheet.create({
  card: { backgroundColor: colors.dark[700], borderRadius: borderRadius.lg, padding: spacing.base, borderWidth: 1, borderColor: colors.dark[600] },
});
```

- [ ] **Step 2: Create Input**

```tsx
// components/ui/Input.tsx
import React, { useState } from 'react';
import { View, TextInput, Text, StyleSheet } from 'react-native';
import { colors } from '../../theme/colors';
import { typography } from '../../theme/typography';
import { borderRadius, spacing } from '../../theme/spacing';

interface InputProps {
  label: string; value: string; onChangeText: (t: string) => void;
  placeholder?: string; secureTextEntry?: boolean; error?: string;
  keyboardType?: 'default' | 'email-address' | 'numeric';
}

export function Input({ label, value, onChangeText, placeholder, secureTextEntry, error, keyboardType = 'default' }: InputProps) {
  const [focused, setFocused] = useState(false);
  return (
    <View style={styles.container}>
      <Text style={styles.label}>{label}</Text>
      <TextInput
        style={[styles.input, focused && styles.focused, error && styles.errorBorder]}
        value={value} onChangeText={onChangeText} placeholder={placeholder}
        placeholderTextColor={colors.text.muted} secureTextEntry={secureTextEntry}
        keyboardType={keyboardType} onFocus={() => setFocused(true)} onBlur={() => setFocused(false)}
        selectionColor={colors.primary[400]}
      />
      {error && <Text style={styles.errorText}>{error}</Text>}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { marginBottom: spacing.base },
  label: { color: colors.text.secondary, fontSize: typography.sizes.sm, fontFamily: typography.fonts.bodyMedium, marginBottom: spacing.xs },
  input: { height: 50, backgroundColor: colors.dark[700], borderRadius: borderRadius.md, paddingHorizontal: spacing.base, color: colors.white, fontSize: typography.sizes.base, fontFamily: typography.fonts.body, borderWidth: 1, borderColor: colors.dark[600] },
  focused: { borderColor: colors.primary[400] },
  errorBorder: { borderColor: colors.error },
  errorText: { color: colors.error, fontSize: typography.sizes.xs, marginTop: spacing.xs },
});
```

- [ ] **Step 3: Create Badge**

```tsx
// components/ui/Badge.tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { colors } from '../../theme/colors';
import { typography } from '../../theme/typography';
import { borderRadius, spacing } from '../../theme/spacing';

interface BadgeProps { label: string; color?: string; bgColor?: string }

export function Badge({ label, color = colors.primary[400], bgColor }: BadgeProps) {
  return (
    <View style={[styles.badge, { backgroundColor: bgColor || `${color}20` }]}>
      <Text style={[styles.text, { color }]}>{label}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  badge: { paddingHorizontal: spacing.sm, paddingVertical: spacing.xs, borderRadius: borderRadius.full, alignSelf: 'flex-start' },
  text: { fontSize: typography.sizes.xs, fontFamily: typography.fonts.bodySemiBold, textTransform: 'uppercase' },
});
```

- [ ] **Step 4: Create Avatar**

```tsx
// components/ui/Avatar.tsx
import React from 'react';
import { View, Image, Text, StyleSheet } from 'react-native';
import { colors } from '../../theme/colors';
import { typography } from '../../theme/typography';

interface AvatarProps { uri?: string; name: string; size?: number }

export function Avatar({ uri, name, size = 48 }: AvatarProps) {
  const initials = name.split(' ').map(w => w[0]).join('').slice(0, 2).toUpperCase();
  return uri ? (
    <Image source={{ uri }} style={[styles.image, { width: size, height: size, borderRadius: size / 2 }]} />
  ) : (
    <View style={[styles.fallback, { width: size, height: size, borderRadius: size / 2 }]}>
      <Text style={[styles.initials, { fontSize: size * 0.38 }]}>{initials}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  image: { backgroundColor: colors.dark[600] },
  fallback: { backgroundColor: colors.primary[800], alignItems: 'center', justifyContent: 'center' },
  initials: { color: colors.primary[400], fontFamily: typography.fonts.heading },
});
```

- [ ] **Step 5: Commit**

```bash
git add components/ui/
git commit -m "feat: add Card, Input, Badge, Avatar UI primitives"
```

---

### Task 7: Navigation Shell with Role-Based Routing

**Files:**
- Create: `app/(auth)/_layout.tsx`
- Create: `app/(auth)/login.tsx` (placeholder screen)
- Create: `app/(player)/_layout.tsx`
- Create: `app/(player)/home.tsx` (placeholder screen)
- Create: `app/(scout)/_layout.tsx`
- Create: `app/(scout)/dashboard.tsx` (placeholder screen)
- Modify: `app/index.tsx`

- [ ] **Step 1: Create auth layout**

```tsx
// app/(auth)/_layout.tsx
import { Stack } from 'expo-router';
import { colors } from '../../theme/colors';

export default function AuthLayout() {
  return <Stack screenOptions={{ headerShown: false, contentStyle: { backgroundColor: colors.dark[800] } }} />;
}
```

- [ ] **Step 2: Create player tab layout**

```tsx
// app/(player)/_layout.tsx
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';
import { colors } from '../../theme/colors';
import { typography } from '../../theme/typography';

export default function PlayerLayout() {
  return (
    <Tabs screenOptions={{
      headerShown: false,
      tabBarStyle: { backgroundColor: colors.dark[900], borderTopColor: colors.dark[700], height: 85, paddingBottom: 25 },
      tabBarActiveTintColor: colors.primary[400],
      tabBarInactiveTintColor: colors.text.muted,
      tabBarLabelStyle: { fontFamily: typography.fonts.bodyMedium, fontSize: 11 },
    }}>
      <Tabs.Screen name="home" options={{ title: 'Home', tabBarIcon: ({ color, size }) => <Ionicons name="home" size={size} color={color} /> }} />
      <Tabs.Screen name="upload" options={{ title: 'Upload', tabBarIcon: ({ color, size }) => <Ionicons name="cloud-upload" size={size} color={color} /> }} />
      <Tabs.Screen name="analysis" options={{ title: 'Analysis', tabBarIcon: ({ color, size }) => <Ionicons name="analytics" size={size} color={color} /> }} />
      <Tabs.Screen name="profile" options={{ title: 'Profile', tabBarIcon: ({ color, size }) => <Ionicons name="person" size={size} color={color} /> }} />
    </Tabs>
  );
}
```

- [ ] **Step 3: Create scout tab layout**

```tsx
// app/(scout)/_layout.tsx
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';
import { colors } from '../../theme/colors';
import { typography } from '../../theme/typography';

export default function ScoutLayout() {
  return (
    <Tabs screenOptions={{
      headerShown: false,
      tabBarStyle: { backgroundColor: colors.dark[900], borderTopColor: colors.dark[700], height: 85, paddingBottom: 25 },
      tabBarActiveTintColor: colors.primary[400],
      tabBarInactiveTintColor: colors.text.muted,
      tabBarLabelStyle: { fontFamily: typography.fonts.bodyMedium, fontSize: 11 },
    }}>
      <Tabs.Screen name="dashboard" options={{ title: 'Dashboard', tabBarIcon: ({ color, size }) => <Ionicons name="stats-chart" size={size} color={color} /> }} />
      <Tabs.Screen name="players" options={{ title: 'Players', tabBarIcon: ({ color, size }) => <Ionicons name="people" size={size} color={color} /> }} />
      <Tabs.Screen name="compare" options={{ title: 'Compare', tabBarIcon: ({ color, size }) => <Ionicons name="git-compare" size={size} color={color} /> }} />
    </Tabs>
  );
}
```

- [ ] **Step 4: Create entry redirect**

```tsx
// app/index.tsx
import { Redirect } from 'expo-router';

export default function Index() {
  // Will be replaced with auth state check in Plan 02
  return <Redirect href="/(auth)/login" />;
}
```

- [ ] **Step 5: Create placeholder screens** — create minimal screens for `login.tsx`, `home.tsx`, `dashboard.tsx` each rendering the screen name in center.

- [ ] **Step 6: Verify navigation** — run app, confirm it redirects to login. Manually test navigating to player/scout tabs via URL.

- [ ] **Step 7: Commit**

```bash
git add app/
git commit -m "feat: add navigation shell with auth/player/scout layouts"
```

---

### Task 8: Supabase Client & React Query Setup

**Files:**
- Create: `lib/supabase.ts`
- Create: `lib/queryClient.ts`
- Create: `lib/constants.ts`
- Modify: `app/_layout.tsx`

- [ ] **Step 1: Create constants**

```typescript
// lib/constants.ts
export const SUPABASE_URL = process.env.EXPO_PUBLIC_SUPABASE_URL || '';
export const SUPABASE_ANON_KEY = process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY || '';
export const VIDEO_MAX_DURATION_SEC = 900; // 15 minutes
export const VIDEO_MAX_SIZE_MB = 500;
```

- [ ] **Step 2: Create Supabase client**

```typescript
// lib/supabase.ts
import 'react-native-url-polyfill/auto';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { createClient } from '@supabase/supabase-js';
import { SUPABASE_URL, SUPABASE_ANON_KEY } from './constants';

export const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
  auth: {
    storage: AsyncStorage,
    autoRefreshToken: true,
    persistSession: true,
    detectSessionInUrl: false,
  },
});
```

- [ ] **Step 3: Create React Query client**

```typescript
// lib/queryClient.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: { staleTime: 1000 * 60, retry: 2, refetchOnWindowFocus: false },
    mutations: { retry: 1 },
  },
});
```

- [ ] **Step 4: Wrap root layout with QueryClientProvider**

Add `QueryClientProvider` wrapping `<Stack />` in `app/_layout.tsx`.

- [ ] **Step 5: Commit**

```bash
git add lib/ app/_layout.tsx
git commit -m "feat: configure Supabase client and React Query"
```

---

### Task 9: TypeScript Types Foundation

**Files:**
- Create: `types/auth.ts`
- Create: `types/player.ts`
- Create: `types/video.ts`
- Create: `types/analysis.ts`
- Create: `types/scout.ts`

- [ ] **Step 1: Create all type files**

```typescript
// types/auth.ts
export type UserRole = 'player' | 'scout';
export interface Profile { id: string; full_name: string; role: UserRole; avatar_url: string | null; created_at: string; }

// types/player.ts
export type Position = 'GK' | 'CB' | 'LB' | 'RB' | 'CDM' | 'CM' | 'CAM' | 'LW' | 'RW' | 'ST' | 'CF';
export type PreferredFoot = 'left' | 'right' | 'both';
export interface PlayerProfile { id: string; user_id: string; age: number; position: Position; height_cm: number | null; weight_kg: number | null; preferred_foot: PreferredFoot; club_name: string | null; country: string | null; bio: string | null; }

// types/video.ts
export type VideoStatus = 'uploaded' | 'processing' | 'completed' | 'failed';
export interface Video { id: string; user_id: string; storage_path: string; thumbnail_url: string | null; duration_seconds: number | null; file_size_bytes: number | null; status: VideoStatus; uploaded_at: string; }

// types/analysis.ts
export type AnalysisStatus = 'pending' | 'processing' | 'completed' | 'failed';
export interface Analysis { id: string; video_id: string; user_id: string; status: AnalysisStatus; distance_covered: number | null; avg_speed: number | null; sprint_count: number | null; ball_touches: number | null; pass_accuracy: number | null; shot_attempts: number | null; movement_score: number | null; technical_score: number | null; decision_score: number | null; activity_score: number | null; composite_score: number | null; completed_at: string | null; created_at: string; }

// types/scout.ts
export interface PlayerRanking { user_id: string; full_name: string; avatar_url: string | null; position: Position; age: number; club_name: string | null; country: string | null; composite_score: number; total_matches: number; }
export interface FilterOptions { positions: Position[]; ageRange: [number, number]; scoreRange: [number, number]; }
```

- [ ] **Step 2: Commit**

```bash
git add types/
git commit -m "feat: add TypeScript type definitions"
```

---

> **Plan 01 complete.** After executing all 9 tasks, you will have: a bootable Expo app, full design system, navigation shell, Supabase + React Query configured, and all TypeScript types. Continue to `plan-02-authentication.md`.
