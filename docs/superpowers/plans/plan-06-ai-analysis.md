# Plan 06: AI Analysis Integration

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Integrate the AI analysis pipeline — trigger analysis after video upload, simulate processing with realistic delays and status updates, generate mock performance metrics, display rich analysis results with charts, and implement real-time status polling.

**Architecture:** For the prototype, AI analysis is simulated via a Supabase Edge Function that generates realistic mock metrics after a configurable delay. The mobile app polls for status updates and renders results when complete. This architecture is designed to be swapped for real AI (Python FastAPI + YOLO) without changing the mobile app.

**Tech Stack:** Supabase Edge Functions (Deno), React Query (polling), Victory Native (charts), Moti (animations)

---

### Task 1: Analysis Service Layer

**Files:**
- Create: `services/analysisService.ts`

- [ ] **Step 1: Implement analysis service**

```typescript
// services/analysisService.ts
import { supabase } from '../lib/supabase';
import { Analysis } from '../types/analysis';

export const analysisService = {
  async triggerAnalysis(videoId: string, userId: string): Promise<Analysis> {
    const { data, error } = await supabase.from('analyses').insert({
      video_id: videoId,
      user_id: userId,
      status: 'pending',
    }).select().single();
    if (error) throw error;

    // Trigger the edge function (fire-and-forget for prototype)
    supabase.functions.invoke('process-video', {
      body: { analysis_id: data.id, video_id: videoId },
    }).catch(() => {}); // Non-blocking

    return data;
  },

  async getAnalysis(analysisId: string): Promise<Analysis> {
    const { data, error } = await supabase.from('analyses')
      .select('*').eq('id', analysisId).single();
    if (error) throw error;
    return data;
  },

  async getAnalysesForUser(userId: string): Promise<Analysis[]> {
    const { data, error } = await supabase.from('analyses')
      .select('*, videos(storage_path, thumbnail_url, duration_seconds)')
      .eq('user_id', userId)
      .order('created_at', { ascending: false });
    if (error) throw error;
    return data || [];
  },

  async getAnalysisForVideo(videoId: string): Promise<Analysis | null> {
    const { data, error } = await supabase.from('analyses')
      .select('*').eq('video_id', videoId).single();
    if (error && error.code !== 'PGRST116') throw error;
    return data;
  },
};
```

- [ ] **Step 2: Commit**

```bash
git add services/analysisService.ts
git commit -m "feat: add analysis service layer"
```

---

### Task 2: Mock AI Edge Function

**Files:**
- Deploy: Supabase Edge Function `process-video`

- [ ] **Step 1: Deploy edge function** using Supabase MCP or CLI

The edge function simulates AI processing:
1. Receives `analysis_id` and `video_id`
2. Updates status to `processing`
3. Waits 5–15 seconds (simulated processing)
4. Generates random but realistic metrics
5. Calculates composite score using the formula from PRD
6. Updates status to `completed`

```typescript
// Edge function: process-video/index.ts
import "jsr:@supabase/functions-js/edge-runtime.d.ts";
import { createClient } from "jsr:@supabase/supabase-js@2";

Deno.serve(async (req: Request) => {
  const { analysis_id, video_id } = await req.json();
  
  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
  );

  // Mark as processing
  await supabase.from("analyses").update({
    status: "processing",
    processing_started_at: new Date().toISOString(),
  }).eq("id", analysis_id);

  // Simulate processing delay (5-15 seconds)
  await new Promise((r) => setTimeout(r, 5000 + Math.random() * 10000));

  // Generate realistic mock metrics
  const rand = (min: number, max: number) => Math.round((min + Math.random() * (max - min)) * 10) / 10;
  
  const movement = rand(40, 95);
  const technical = rand(35, 90);
  const decision = rand(30, 85);
  const activity = rand(45, 95);
  const composite = Math.round((0.25 * movement + 0.30 * technical + 0.20 * decision + 0.25 * activity) * 10) / 10;

  await supabase.from("analyses").update({
    status: "completed",
    distance_covered: rand(3000, 12000),
    avg_speed: rand(5, 25),
    sprint_count: Math.round(rand(5, 30)),
    ball_touches: Math.round(rand(20, 120)),
    pass_accuracy: rand(45, 95),
    shot_attempts: Math.round(rand(0, 12)),
    movement_score: movement,
    technical_score: technical,
    decision_score: decision,
    activity_score: activity,
    composite_score: composite,
    completed_at: new Date().toISOString(),
  }).eq("id", analysis_id);

  // Update video status
  await supabase.from("videos").update({ status: "completed" }).eq("id", video_id);

  return new Response(JSON.stringify({ success: true, composite_score: composite }), {
    headers: { "Content-Type": "application/json" },
  });
});
```

- [ ] **Step 2: Commit** (if using local edge function files)

---

### Task 3: Analysis Polling Hook

**Files:**
- Create: `hooks/useAnalysis.ts`

- [ ] **Step 1: Implement polling hook**

```typescript
// hooks/useAnalysis.ts
import { useQuery } from '@tanstack/react-query';
import { analysisService } from '../services/analysisService';
import { Analysis } from '../types/analysis';

export function useAnalysis(analysisId: string | null) {
  return useQuery<Analysis>({
    queryKey: ['analysis', analysisId],
    queryFn: () => analysisService.getAnalysis(analysisId!),
    enabled: !!analysisId,
    refetchInterval: (query) => {
      const status = query.state.data?.status;
      if (status === 'completed' || status === 'failed') return false;
      return 3000; // Poll every 3 seconds while processing
    },
  });
}

export function useUserAnalyses(userId: string | null) {
  return useQuery<Analysis[]>({
    queryKey: ['analyses', userId],
    queryFn: () => analysisService.getAnalysesForUser(userId!),
    enabled: !!userId,
  });
}
```

- [ ] **Step 2: Commit**

```bash
git add hooks/useAnalysis.ts
git commit -m "feat: add analysis polling hook"
```

---

### Task 4: Analysis Result Screen

**Files:**
- Create: `app/(player)/analysis/[id].tsx`
- Create: `app/(player)/analysis/index.tsx`
- Create: `components/ui/RadarChart.tsx`

- [ ] **Step 1: Create RadarChart** — SVG-based radar/spider chart showing 4 axes (Movement, Technical, Decision, Activity) with the player's scores plotted. Use `react-native-svg` for drawing.

- [ ] **Step 2: Create analysis detail screen** `[id].tsx`:
  - Shows processing animation (Lottie or pulsing dots) while status is pending/processing
  - On completion: composite score hero number (animated count-up), RadarChart, stat cards grid (distance, speed, sprints, touches, pass accuracy, shots), sub-score breakdown with MetricRows
  - Share button (future) and "Upload Another" CTA

- [ ] **Step 3: Create analysis list screen** `index.tsx` — chronological list of all analyses using AnalysisCard components. Tapping navigates to `[id]`.

- [ ] **Step 4: Verify** — upload video → trigger analysis → see processing state → poll completes → view results.

- [ ] **Step 5: Commit**

```bash
git add app/(player)/analysis/ components/ui/RadarChart.tsx
git commit -m "feat: add analysis result screens with radar chart"
```

---

### Task 5: Connect Upload → Analysis Flow

**Files:**
- Modify: `app/(player)/upload.tsx`
- Modify: `hooks/useVideoUpload.ts`

- [ ] **Step 1: Update upload hook** — after `saveVideoMetadata` succeeds, automatically call `analysisService.triggerAnalysis(video.id, user.id)`.

- [ ] **Step 2: Update upload screen** — on upload complete, show "Analysis Started" state with animated indicator, then CTA "View Analysis" that navigates to `/(player)/analysis/[id]`.

- [ ] **Step 3: Verify end-to-end** — upload → auto-trigger → navigate to analysis → see polling → results appear.

- [ ] **Step 4: Commit**

```bash
git add app/(player)/upload.tsx hooks/useVideoUpload.ts
git commit -m "feat: connect upload to analysis pipeline"
```

---

### Task 6: Polish & Animations

**Files:**
- Various component files

- [ ] **Step 1: Add entrance animations** — use Moti's `MotiView` with `from={{ opacity: 0, translateY: 20 }}` and `animate={{ opacity: 1, translateY: 0 }}` on main screen containers and card lists with staggered delays.

- [ ] **Step 2: Add score count-up animation** — animate composite score from 0 to final value over 1.5s on the analysis result screen using `useSharedValue` + `withTiming` from Reanimated.

- [ ] **Step 3: Add skeleton loaders** — create `SkeletonLoader` component using Reanimated shimmer effect, use on all screens during `isLoading` states.

- [ ] **Step 4: Commit**

```bash
git add components/
git commit -m "feat: add entrance animations, score counter, and skeletons"
```

---

### Task 7: Final Integration Test & Build

- [ ] **Step 1: Run all tests**

```bash
npm test
```
Expected: All tests pass.

- [ ] **Step 2: TypeScript check**

```bash
npx tsc --noEmit
```
Expected: No errors.

- [ ] **Step 3: Test complete flows**
  - Player: signup → profile setup → upload video → view analysis → check profile
  - Scout: signup → dashboard → filter players → view detail → compare two players

- [ ] **Step 4: Create preview build**

```bash
npx eas build --platform ios --profile preview
```

- [ ] **Step 5: Final commit**

```bash
git add -A
git commit -m "feat: complete HiddenStar V2 prototype"
```

---

> **Plan 06 complete.** All 6 plans fully executed = complete working prototype of the AI Football Talent Detection app. 🏟️⚽
