# Plan 04: Player Profile & Data Layer

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build the player profile system — profile setup form, profile display, match history, performance stats cards, and the player data service layer.

**Architecture:** Player data lives in `player_profiles` table linked to `profiles`. `playerService` handles CRUD. Profile screen shows personal info + aggregated stats from analyses. Match history lists videos with their analysis status.

**Tech Stack:** Supabase, React Query, Zustand, React Native

---

### Task 1: Player Service Layer

**Files:**
- Create: `services/playerService.ts`

- [ ] **Step 1: Implement player service**

```typescript
// services/playerService.ts
import { supabase } from '../lib/supabase';
import { PlayerProfile, Position, PreferredFoot } from '../types/player';

export const playerService = {
  async createPlayerProfile(data: {
    user_id: string; age: number; position: Position;
    height_cm?: number; weight_kg?: number; preferred_foot: PreferredFoot;
    club_name?: string; country?: string; bio?: string;
  }): Promise<PlayerProfile> {
    const { data: profile, error } = await supabase.from('player_profiles')
      .insert(data).select().single();
    if (error) throw error;
    return profile;
  },

  async getPlayerProfile(userId: string): Promise<PlayerProfile | null> {
    const { data, error } = await supabase.from('player_profiles')
      .select('*').eq('user_id', userId).single();
    if (error && error.code !== 'PGRST116') throw error;
    return data;
  },

  async updatePlayerProfile(userId: string, updates: Partial<PlayerProfile>): Promise<PlayerProfile> {
    const { data, error } = await supabase.from('player_profiles')
      .update(updates).eq('user_id', userId).select().single();
    if (error) throw error;
    return data;
  },

  async getPlayerStats(userId: string) {
    const { data, error } = await supabase.from('analyses')
      .select('composite_score, movement_score, technical_score, decision_score, activity_score, distance_covered, sprint_count, pass_accuracy, completed_at')
      .eq('user_id', userId).eq('status', 'completed').order('completed_at', { ascending: false });
    if (error) throw error;
    return data || [];
  },

  async getPlayerWithLatestStats(userId: string) {
    const [profile, stats] = await Promise.all([
      this.getPlayerProfile(userId),
      this.getPlayerStats(userId),
    ]);
    const latest = stats[0] || null;
    const avgScore = stats.length > 0
      ? stats.reduce((sum, s) => sum + (s.composite_score || 0), 0) / stats.length
      : 0;
    return { profile, latestAnalysis: latest, averageScore: Math.round(avgScore * 10) / 10, totalMatches: stats.length };
  },
};
```

- [ ] **Step 2: Commit**

```bash
git add services/playerService.ts
git commit -m "feat: add player service layer"
```

---

### Task 2: Player Profile Setup Screen

**Files:**
- Create: `app/(player)/profile-setup.tsx`

- [ ] **Step 1: Build profile setup form** — collects age, position (picker), height, weight, preferred foot, club name, country. Uses `Input` and `Button` components. On submit, calls `playerService.createPlayerProfile()`.

- [ ] **Step 2: Add navigation** — after role-select as player, if no player_profile exists, redirect here before home.

- [ ] **Step 3: Verify** — complete signup as player → lands on setup → fill form → navigates to home.

- [ ] **Step 4: Commit**

```bash
git add app/(player)/profile-setup.tsx
git commit -m "feat: add player profile setup screen"
```

---

### Task 3: Performance Overview Components

**Files:**
- Create: `components/ui/StatCard.tsx`
- Create: `components/player/PerformanceOverview.tsx`
- Create: `components/player/MetricRow.tsx`

- [ ] **Step 1: Create StatCard**

```tsx
// components/ui/StatCard.tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { colors } from '../../theme/colors';
import { typography } from '../../theme/typography';
import { spacing, borderRadius } from '../../theme/spacing';

interface StatCardProps { label: string; value: string | number; unit?: string; color?: string }

export function StatCard({ label, value, unit, color = colors.primary[400] }: StatCardProps) {
  return (
    <View style={styles.card}>
      <Text style={[styles.value, { color }]}>{value}{unit && <Text style={styles.unit}>{unit}</Text>}</Text>
      <Text style={styles.label}>{label}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  card: { backgroundColor: colors.dark[700], borderRadius: borderRadius.lg, padding: spacing.base, flex: 1, alignItems: 'center', borderWidth: 1, borderColor: colors.dark[600] },
  value: { fontFamily: typography.fonts.heading, fontSize: typography.sizes['2xl'] },
  unit: { fontSize: typography.sizes.sm, color: colors.text.muted },
  label: { fontFamily: typography.fonts.body, fontSize: typography.sizes.xs, color: colors.text.secondary, marginTop: spacing.xs },
});
```

- [ ] **Step 2: Create MetricRow** — horizontal row displaying metric name, value, and a mini progress bar (0–100 scale).

- [ ] **Step 3: Create PerformanceOverview** — grid of StatCards (Overall Score, Matches Played, Avg Speed, Total Distance) + list of MetricRows for each sub-score (Movement, Technical, Decision, Activity).

- [ ] **Step 4: Commit**

```bash
git add components/ui/StatCard.tsx components/player/PerformanceOverview.tsx components/player/MetricRow.tsx
git commit -m "feat: add performance overview components"
```

---

### Task 4: Match History Components

**Files:**
- Create: `components/player/MatchHistoryItem.tsx`
- Create: `components/player/AnalysisCard.tsx`

- [ ] **Step 1: Create MatchHistoryItem** — card showing video thumbnail, date, duration, and analysis status badge (pending/processing/completed/failed).

- [ ] **Step 2: Create AnalysisCard** — tappable card linking to analysis detail, shows composite score as large number with color coding (green >70, amber >40, red <40).

- [ ] **Step 3: Commit**

```bash
git add components/player/MatchHistoryItem.tsx components/player/AnalysisCard.tsx
git commit -m "feat: add match history and analysis card components"
```

---

### Task 5: Player Home Screen

**Files:**
- Modify: `app/(player)/home.tsx`

- [ ] **Step 1: Build home screen** — greeting header with avatar + name, PerformanceOverview (latest stats), recent match history (last 5), and a "Upload New Match" CTA button.

- [ ] **Step 2: Use React Query** — `useQuery` for player stats and video list, with pull-to-refresh via `RefreshControl`.

- [ ] **Step 3: Verify** — run app as player, confirm home shows stats and history (empty states if no data).

- [ ] **Step 4: Commit**

```bash
git add app/(player)/home.tsx
git commit -m "feat: build player home screen with stats and history"
```

---

### Task 6: Player Profile Screen

**Files:**
- Modify: `app/(player)/profile.tsx`

- [ ] **Step 1: Build profile screen** — avatar, name, role badge, player details (position, age, height, weight, foot, club), edit profile button, sign out button.

- [ ] **Step 2: Add edit modal** — bottom sheet or modal with pre-filled Input fields to update player_profile.

- [ ] **Step 3: Verify** — view profile, tap edit, change a field, save, confirm update.

- [ ] **Step 4: Commit**

```bash
git add app/(player)/profile.tsx
git commit -m "feat: build player profile screen with edit"
```

---

### Task 7: Score Calculator Utility

**Files:**
- Create: `utils/scoreCalculator.ts`
- Test: `__tests__/utils/scoreCalculator.test.ts`

- [ ] **Step 1: Write the test**

```typescript
// __tests__/utils/scoreCalculator.test.ts
import { calculateCompositeScore, getScoreColor, getScoreLabel } from '../../utils/scoreCalculator';

describe('scoreCalculator', () => {
  it('calculates weighted composite score', () => {
    const score = calculateCompositeScore({ movement: 80, technical: 70, decision: 60, activity: 90 });
    // (0.25*80) + (0.30*70) + (0.20*60) + (0.25*90) = 20+21+12+22.5 = 75.5
    expect(score).toBe(75.5);
  });
  it('returns green for score > 70', () => {
    expect(getScoreColor(75)).toBe('#00C853');
  });
  it('returns amber for score 40-70', () => {
    expect(getScoreColor(55)).toBe('#FF9500');
  });
  it('returns red for score < 40', () => {
    expect(getScoreColor(30)).toBe('#FF3B30');
  });
});
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Implement**

```typescript
// utils/scoreCalculator.ts
import { colors } from '../theme/colors';

interface SubScores { movement: number; technical: number; decision: number; activity: number }

export function calculateCompositeScore(scores: SubScores): number {
  return Math.round(
    (0.25 * scores.movement + 0.30 * scores.technical + 0.20 * scores.decision + 0.25 * scores.activity) * 10
  ) / 10;
}

export function getScoreColor(score: number): string {
  if (score >= 70) return colors.success;
  if (score >= 40) return colors.warning;
  return colors.error;
}

export function getScoreLabel(score: number): string {
  if (score >= 85) return 'Elite';
  if (score >= 70) return 'Excellent';
  if (score >= 55) return 'Good';
  if (score >= 40) return 'Average';
  return 'Developing';
}
```

- [ ] **Step 4: Run test — expect PASS**
- [ ] **Step 5: Commit**

```bash
git add utils/scoreCalculator.ts __tests__/utils/scoreCalculator.test.ts
git commit -m "feat: add score calculator with tests"
```

---

> **Plan 04 complete.** After executing all 7 tasks: complete player data layer, profile setup, home screen with stats, profile screen with edit, match history display, and score calculation. Continue to `plan-05-scout-dashboard.md`.
