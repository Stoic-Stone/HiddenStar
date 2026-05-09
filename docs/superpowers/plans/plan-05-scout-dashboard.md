# Plan 05: Scout Dashboard & Discovery

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build the scout experience — dashboard with player rankings, advanced filters (position, age, score range), player detail view, search, and side-by-side player comparison.

**Architecture:** Scout queries join `profiles`, `player_profiles`, and `analyses` tables to produce ranked player lists. Filters are managed in Zustand `scoutStore`. Comparison view allows selecting 2 players and rendering their metrics side-by-side with radar charts.

**Tech Stack:** Supabase (complex queries), Zustand, React Query, Victory Native (radar chart)

---

### Task 1: Scout Service Layer

**Files:**
- Create: `services/scoutService.ts`

- [ ] **Step 1: Implement scout service**

```typescript
// services/scoutService.ts
import { supabase } from '../lib/supabase';
import { PlayerRanking, FilterOptions } from '../types/scout';

export const scoutService = {
  async getPlayerRankings(filters?: Partial<FilterOptions>): Promise<PlayerRanking[]> {
    let query = supabase.from('analyses')
      .select(`
        user_id,
        composite_score,
        profiles!inner(full_name, avatar_url),
        player_profiles!inner(position, age, club_name, country)
      `)
      .eq('status', 'completed')
      .order('composite_score', { ascending: false });

    if (filters?.positions?.length) {
      query = query.in('player_profiles.position', filters.positions);
    }
    if (filters?.ageRange) {
      query = query.gte('player_profiles.age', filters.ageRange[0])
        .lte('player_profiles.age', filters.ageRange[1]);
    }
    if (filters?.scoreRange) {
      query = query.gte('composite_score', filters.scoreRange[0])
        .lte('composite_score', filters.scoreRange[1]);
    }

    const { data, error } = await query;
    if (error) throw error;

    // Aggregate: group by user, take best score, count matches
    const grouped = new Map<string, PlayerRanking>();
    (data || []).forEach((row: any) => {
      const existing = grouped.get(row.user_id);
      if (!existing || row.composite_score > existing.composite_score) {
        grouped.set(row.user_id, {
          user_id: row.user_id,
          full_name: row.profiles.full_name,
          avatar_url: row.profiles.avatar_url,
          position: row.player_profiles.position,
          age: row.player_profiles.age,
          club_name: row.player_profiles.club_name,
          country: row.player_profiles.country,
          composite_score: row.composite_score,
          total_matches: (existing?.total_matches || 0) + 1,
        });
      } else if (existing) {
        existing.total_matches += 1;
      }
    });

    return Array.from(grouped.values()).sort((a, b) => b.composite_score - a.composite_score);
  },

  async getPlayerDetail(userId: string) {
    const [profileRes, playerRes, analysesRes] = await Promise.all([
      supabase.from('profiles').select('*').eq('id', userId).single(),
      supabase.from('player_profiles').select('*').eq('user_id', userId).single(),
      supabase.from('analyses').select('*').eq('user_id', userId).eq('status', 'completed')
        .order('completed_at', { ascending: false }),
    ]);
    if (profileRes.error) throw profileRes.error;
    return {
      profile: profileRes.data,
      playerProfile: playerRes.data,
      analyses: analysesRes.data || [],
    };
  },

  async searchPlayers(query: string): Promise<PlayerRanking[]> {
    const { data, error } = await supabase.from('profiles')
      .select('id, full_name, avatar_url, player_profiles(position, age, club_name, country)')
      .eq('role', 'player')
      .ilike('full_name', `%${query}%`)
      .limit(20);
    if (error) throw error;
    return (data || []).map((row: any) => ({
      user_id: row.id,
      full_name: row.full_name,
      avatar_url: row.avatar_url,
      position: row.player_profiles?.position || 'ST',
      age: row.player_profiles?.age || 0,
      club_name: row.player_profiles?.club_name,
      country: row.player_profiles?.country,
      composite_score: 0,
      total_matches: 0,
    }));
  },
};
```

- [ ] **Step 2: Commit**

```bash
git add services/scoutService.ts
git commit -m "feat: add scout service with rankings, detail, and search"
```

---

### Task 2: Scout Zustand Store

**Files:**
- Create: `stores/scoutStore.ts`

- [ ] **Step 1: Implement scout store**

```typescript
// stores/scoutStore.ts
import { create } from 'zustand';
import { FilterOptions, PlayerRanking } from '../types/scout';
import { Position } from '../types/player';

interface ScoutState {
  filters: Partial<FilterOptions>;
  comparisonIds: [string | null, string | null];
  setFilter: <K extends keyof FilterOptions>(key: K, value: FilterOptions[K]) => void;
  clearFilters: () => void;
  setComparisonPlayer: (slot: 0 | 1, userId: string | null) => void;
  clearComparison: () => void;
}

export const useScoutStore = create<ScoutState>((set, get) => ({
  filters: {},
  comparisonIds: [null, null],
  setFilter: (key, value) => set((s) => ({ filters: { ...s.filters, [key]: value } })),
  clearFilters: () => set({ filters: {} }),
  setComparisonPlayer: (slot, userId) => set((s) => {
    const ids = [...s.comparisonIds] as [string | null, string | null];
    ids[slot] = userId;
    return { comparisonIds: ids };
  }),
  clearComparison: () => set({ comparisonIds: [null, null] }),
}));
```

- [ ] **Step 2: Commit**

```bash
git add stores/scoutStore.ts
git commit -m "feat: add scout Zustand store"
```

---

### Task 3: Scout UI Components

**Files:**
- Create: `components/scout/PlayerRankingCard.tsx`
- Create: `components/scout/FilterSheet.tsx`
- Create: `components/scout/PlayerSearchBar.tsx`

- [ ] **Step 1: Create PlayerRankingCard** — horizontal card with rank number, avatar, name, position badge, age, composite score (large, color-coded), chevron icon. Tappable → navigates to player detail.

- [ ] **Step 2: Create FilterSheet** — bottom sheet modal with position multi-select chips, age range slider (12–18), score range slider (0–100), apply + clear buttons.

- [ ] **Step 3: Create PlayerSearchBar** — styled text input with search icon, debounced (300ms) query, calls `scoutService.searchPlayers`.

- [ ] **Step 4: Commit**

```bash
git add components/scout/
git commit -m "feat: add scout UI components"
```

---

### Task 4: Scout Dashboard Screen

**Files:**
- Modify: `app/(scout)/dashboard.tsx`

- [ ] **Step 1: Build dashboard** — header with greeting + total players count, top 3 players highlight cards (podium style), "View All Rankings" CTA, recent activity feed placeholder.

- [ ] **Step 2: Use React Query** — `useQuery(['rankings'], () => scoutService.getPlayerRankings())` with pull-to-refresh.

- [ ] **Step 3: Verify** — run app as scout, confirm dashboard renders.

- [ ] **Step 4: Commit**

```bash
git add app/(scout)/dashboard.tsx
git commit -m "feat: build scout dashboard screen"
```

---

### Task 5: Player List & Detail Screens

**Files:**
- Create: `app/(scout)/players/index.tsx`
- Create: `app/(scout)/players/[id].tsx`

- [ ] **Step 1: Build player list** — FlatList of PlayerRankingCards, search bar at top, filter icon opens FilterSheet, empty state for no results.

- [ ] **Step 2: Build player detail** — full player profile view for scouts: avatar, name, position, age, club, physical stats, all-time metrics, match-by-match score chart, "Add to Comparison" button.

- [ ] **Step 3: Verify** — tap a player in list → opens detail with full stats.

- [ ] **Step 4: Commit**

```bash
git add app/(scout)/players/
git commit -m "feat: add player list and detail screens for scout"
```

---

### Task 6: Player Comparison Screen

**Files:**
- Modify: `app/(scout)/compare.tsx`
- Create: `components/scout/ComparisonTable.tsx`

- [ ] **Step 1: Build ComparisonTable** — two columns, each showing a player's metrics. Rows: composite score, movement, technical, decision, activity, distance, sprints, pass accuracy. Color-highlight the higher value in each row.

- [ ] **Step 2: Build compare screen** — two player selector slots (tap to search/pick), once both filled show ComparisonTable + radar chart overlay.

- [ ] **Step 3: Verify** — select two players, confirm side-by-side comparison renders.

- [ ] **Step 4: Commit**

```bash
git add app/(scout)/compare.tsx components/scout/ComparisonTable.tsx
git commit -m "feat: add player comparison screen"
```

---

> **Plan 05 complete.** After executing all 6 tasks: full scout experience with dashboard, ranked player list, filters, search, player detail, and side-by-side comparison. Continue to `plan-06-ai-analysis.md`.
