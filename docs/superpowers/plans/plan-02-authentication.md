# Plan 02: Authentication & Onboarding

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement complete auth flow — email signup/login, Google/Apple social auth, role selection (Player/Scout), profile creation, and session-based route guarding.

**Architecture:** Supabase Auth handles identity. On signup, user selects role, which creates a `profiles` row. Auth state is managed in Zustand `authStore` and drives navigation: unauthenticated → auth stack, player → player tabs, scout → scout tabs.

**Tech Stack:** Supabase Auth, Zustand, Expo Router, expo-auth-session, expo-apple-authentication

---

### Task 1: Auth Zustand Store

**Files:**
- Create: `stores/authStore.ts`
- Test: `__tests__/stores/authStore.test.ts`

- [ ] **Step 1: Write the test**

```typescript
// __tests__/stores/authStore.test.ts
import { useAuthStore } from '../../stores/authStore';

describe('authStore', () => {
  beforeEach(() => useAuthStore.getState().reset());

  it('starts with null user and no session', () => {
    const state = useAuthStore.getState();
    expect(state.user).toBeNull();
    expect(state.session).toBeNull();
    expect(state.isAuthenticated).toBe(false);
  });

  it('setSession updates auth state', () => {
    const mockSession = { access_token: 'abc', user: { id: '123', email: 'test@mail.com' } };
    useAuthStore.getState().setSession(mockSession as any);
    expect(useAuthStore.getState().isAuthenticated).toBe(true);
  });

  it('setProfile stores profile with role', () => {
    useAuthStore.getState().setProfile({ id: '123', full_name: 'Ali', role: 'player', avatar_url: null, created_at: '' });
    expect(useAuthStore.getState().profile?.role).toBe('player');
  });
});
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Implement authStore**

```typescript
// stores/authStore.ts
import { create } from 'zustand';
import { Session, User } from '@supabase/supabase-js';
import { Profile } from '../types/auth';

interface AuthState {
  session: Session | null;
  user: User | null;
  profile: Profile | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  setSession: (session: Session | null) => void;
  setProfile: (profile: Profile | null) => void;
  setLoading: (loading: boolean) => void;
  reset: () => void;
}

export const useAuthStore = create<AuthState>((set) => ({
  session: null,
  user: null,
  profile: null,
  isAuthenticated: false,
  isLoading: true,
  setSession: (session) => set({
    session,
    user: session?.user ?? null,
    isAuthenticated: !!session,
  }),
  setProfile: (profile) => set({ profile }),
  setLoading: (isLoading) => set({ isLoading }),
  reset: () => set({ session: null, user: null, profile: null, isAuthenticated: false, isLoading: false }),
}));
```

- [ ] **Step 4: Run test — expect PASS**
- [ ] **Step 5: Commit**

```bash
git add stores/authStore.ts __tests__/stores/authStore.test.ts
git commit -m "feat: add auth Zustand store"
```

---

### Task 2: Auth Service Layer

**Files:**
- Create: `services/authService.ts`

- [ ] **Step 1: Implement auth service**

```typescript
// services/authService.ts
import { supabase } from '../lib/supabase';
import { Profile, UserRole } from '../types/auth';

export const authService = {
  async signUpWithEmail(email: string, password: string) {
    const { data, error } = await supabase.auth.signUp({ email, password });
    if (error) throw error;
    return data;
  },

  async signInWithEmail(email: string, password: string) {
    const { data, error } = await supabase.auth.signInWithPassword({ email, password });
    if (error) throw error;
    return data;
  },

  async signOut() {
    const { error } = await supabase.auth.signOut();
    if (error) throw error;
  },

  async getSession() {
    const { data: { session }, error } = await supabase.auth.getSession();
    if (error) throw error;
    return session;
  },

  async createProfile(userId: string, fullName: string, role: UserRole): Promise<Profile> {
    const { data, error } = await supabase.from('profiles').insert({
      id: userId, full_name: fullName, role,
    }).select().single();
    if (error) throw error;
    return data;
  },

  async getProfile(userId: string): Promise<Profile | null> {
    const { data, error } = await supabase.from('profiles').select('*').eq('id', userId).single();
    if (error && error.code !== 'PGRST116') throw error;
    return data;
  },
};
```

- [ ] **Step 2: Commit**

```bash
git add services/authService.ts
git commit -m "feat: add auth service layer"
```

---

### Task 3: Login Screen

**Files:**
- Create: `app/(auth)/login.tsx`
- Create: `components/auth/LoginForm.tsx`

- [ ] **Step 1: Build LoginForm component**

```tsx
// components/auth/LoginForm.tsx
import React, { useState } from 'react';
import { View, Text, StyleSheet, Alert } from 'react-native';
import { Input } from '../ui/Input';
import { Button } from '../ui/Button';
import { colors } from '../../theme/colors';
import { typography } from '../../theme/typography';
import { spacing } from '../../theme/spacing';
import { authService } from '../../services/authService';
import { useAuthStore } from '../../stores/authStore';

export function LoginForm({ onNavigateRegister }: { onNavigateRegister: () => void }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);
  const setSession = useAuthStore((s) => s.setSession);

  const handleLogin = async () => {
    if (!email || !password) { Alert.alert('Error', 'Please fill all fields'); return; }
    setLoading(true);
    try {
      const { session } = await authService.signInWithEmail(email, password);
      setSession(session);
    } catch (err: any) {
      Alert.alert('Login Failed', err.message);
    } finally { setLoading(false); }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Welcome Back</Text>
      <Text style={styles.subtitle}>Sign in to continue your journey</Text>
      <Input label="Email" value={email} onChangeText={setEmail} placeholder="you@email.com" keyboardType="email-address" />
      <Input label="Password" value={password} onChangeText={setPassword} placeholder="••••••••" secureTextEntry />
      <Button label="Sign In" onPress={handleLogin} loading={loading} fullWidth />
      <Button label="Create Account" onPress={onNavigateRegister} variant="ghost" fullWidth />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', paddingHorizontal: spacing.xl },
  title: { fontFamily: typography.fonts.heading, fontSize: typography.sizes['3xl'], color: colors.white, marginBottom: spacing.xs },
  subtitle: { fontFamily: typography.fonts.body, fontSize: typography.sizes.base, color: colors.text.secondary, marginBottom: spacing['2xl'] },
});
```

- [ ] **Step 2: Build login screen**

```tsx
// app/(auth)/login.tsx
import React from 'react';
import { SafeAreaView, StyleSheet } from 'react-native';
import { router } from 'expo-router';
import { LoginForm } from '../../components/auth/LoginForm';
import { colors } from '../../theme/colors';

export default function LoginScreen() {
  return (
    <SafeAreaView style={styles.container}>
      <LoginForm onNavigateRegister={() => router.push('/(auth)/register')} />
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: colors.dark[800] },
});
```

- [ ] **Step 3: Verify screen renders** — run app, confirm login form appears
- [ ] **Step 4: Commit**

```bash
git add app/(auth)/login.tsx components/auth/LoginForm.tsx
git commit -m "feat: add login screen with form"
```

---

### Task 4: Register Screen

**Files:**
- Create: `app/(auth)/register.tsx`
- Create: `components/auth/RegisterForm.tsx`

- [ ] **Step 1: Build RegisterForm** — similar structure to LoginForm but with name field, confirm password, and calls `authService.signUpWithEmail`.

- [ ] **Step 2: Build register screen** — wraps RegisterForm, navigates to role-select on success.

- [ ] **Step 3: Commit**

```bash
git add app/(auth)/register.tsx components/auth/RegisterForm.tsx
git commit -m "feat: add register screen"
```

---

### Task 5: Role Selection Screen

**Files:**
- Create: `app/(auth)/role-select.tsx`

- [ ] **Step 1: Build role selection screen**

```tsx
// app/(auth)/role-select.tsx
import React from 'react';
import { View, Text, TouchableOpacity, StyleSheet, SafeAreaView } from 'react-native';
import { Ionicons } from '@expo/vector-icons';
import { router } from 'expo-router';
import { colors } from '../../theme/colors';
import { typography } from '../../theme/typography';
import { spacing, borderRadius } from '../../theme/spacing';
import { useAuthStore } from '../../stores/authStore';
import { authService } from '../../services/authService';

const roles = [
  { key: 'player' as const, icon: 'football', title: 'Player', desc: 'Upload videos, get scouted, track your performance' },
  { key: 'scout' as const, icon: 'search', title: 'Scout', desc: 'Discover talent, compare players, build your roster' },
];

export default function RoleSelectScreen() {
  const { user, setProfile } = useAuthStore();

  const selectRole = async (role: 'player' | 'scout') => {
    if (!user) return;
    const profile = await authService.createProfile(user.id, user.email || '', role);
    setProfile(profile);
    router.replace(role === 'player' ? '/(player)/home' : '/(scout)/dashboard');
  };

  return (
    <SafeAreaView style={styles.container}>
      <Text style={styles.title}>Who Are You?</Text>
      <Text style={styles.subtitle}>Select your role to personalize your experience</Text>
      <View style={styles.cards}>
        {roles.map((r) => (
          <TouchableOpacity key={r.key} style={styles.card} onPress={() => selectRole(r.key)} activeOpacity={0.85}>
            <Ionicons name={r.icon as any} size={48} color={colors.primary[400]} />
            <Text style={styles.cardTitle}>{r.title}</Text>
            <Text style={styles.cardDesc}>{r.desc}</Text>
          </TouchableOpacity>
        ))}
      </View>
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: colors.dark[800], paddingHorizontal: spacing.xl, justifyContent: 'center' },
  title: { fontFamily: typography.fonts.heading, fontSize: typography.sizes['3xl'], color: colors.white, textAlign: 'center' },
  subtitle: { fontFamily: typography.fonts.body, fontSize: typography.sizes.base, color: colors.text.secondary, textAlign: 'center', marginTop: spacing.sm, marginBottom: spacing['2xl'] },
  cards: { gap: spacing.base },
  card: { backgroundColor: colors.dark[700], borderRadius: borderRadius.xl, padding: spacing['2xl'], alignItems: 'center', borderWidth: 1, borderColor: colors.dark[600] },
  cardTitle: { fontFamily: typography.fonts.heading, fontSize: typography.sizes.xl, color: colors.white, marginTop: spacing.base },
  cardDesc: { fontFamily: typography.fonts.body, fontSize: typography.sizes.sm, color: colors.text.secondary, textAlign: 'center', marginTop: spacing.xs },
});
```

- [ ] **Step 2: Verify flow** — register → lands on role select → tap role → navigates to correct tab
- [ ] **Step 3: Commit**

```bash
git add app/(auth)/role-select.tsx
git commit -m "feat: add role selection screen"
```

---

### Task 6: Auth Session Listener & Route Guard

**Files:**
- Create: `hooks/useAuth.ts`
- Modify: `app/_layout.tsx`
- Modify: `app/index.tsx`

- [ ] **Step 1: Create useAuth hook**

```typescript
// hooks/useAuth.ts
import { useEffect } from 'react';
import { supabase } from '../lib/supabase';
import { useAuthStore } from '../stores/authStore';
import { authService } from '../services/authService';

export function useAuth() {
  const { setSession, setProfile, setLoading } = useAuthStore();

  useEffect(() => {
    supabase.auth.getSession().then(async ({ data: { session } }) => {
      setSession(session);
      if (session?.user) {
        const profile = await authService.getProfile(session.user.id);
        setProfile(profile);
      }
      setLoading(false);
    });

    const { data: { subscription } } = supabase.auth.onAuthStateChange(async (_event, session) => {
      setSession(session);
      if (session?.user) {
        const profile = await authService.getProfile(session.user.id);
        setProfile(profile);
      } else {
        setProfile(null);
      }
    });

    return () => subscription.unsubscribe();
  }, []);
}
```

- [ ] **Step 2: Update root layout** to call `useAuth()` and show loading state.

- [ ] **Step 3: Update index.tsx** to redirect based on auth state and role:

```tsx
// app/index.tsx
import { Redirect } from 'expo-router';
import { useAuthStore } from '../stores/authStore';
import { LoadingOverlay } from '../components/shared/LoadingOverlay';

export default function Index() {
  const { isAuthenticated, profile, isLoading } = useAuthStore();
  if (isLoading) return <LoadingOverlay />;
  if (!isAuthenticated) return <Redirect href="/(auth)/login" />;
  if (!profile) return <Redirect href="/(auth)/role-select" />;
  return <Redirect href={profile.role === 'player' ? '/(player)/home' : '/(scout)/dashboard'} />;
}
```

- [ ] **Step 4: Create LoadingOverlay**

```tsx
// components/shared/LoadingOverlay.tsx
import React from 'react';
import { View, ActivityIndicator, StyleSheet } from 'react-native';
import { colors } from '../../theme/colors';

export function LoadingOverlay() {
  return (
    <View style={styles.container}>
      <ActivityIndicator size="large" color={colors.primary[400]} />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: colors.dark[800], alignItems: 'center', justifyContent: 'center' },
});
```

- [ ] **Step 5: Commit**

```bash
git add hooks/useAuth.ts app/_layout.tsx app/index.tsx components/shared/LoadingOverlay.tsx
git commit -m "feat: add auth listener and role-based route guard"
```

---

> **Plan 02 complete.** After executing all 6 tasks you have: working email auth, role selection, profile creation, session persistence, and automatic role-based navigation. Continue to `plan-03-video-upload.md`.
