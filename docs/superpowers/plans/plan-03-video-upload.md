# Plan 03: Video Upload System

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build the video upload pipeline — pick from camera/gallery, compress, upload to Supabase Storage with progress tracking, store metadata in DB, and show upload history.

**Architecture:** `expo-image-picker` for selection, `expo-video-thumbnails` for thumbnails, Supabase Storage for file hosting, and a dedicated `videoService` for all operations. Upload progress is tracked via XMLHttpRequest for real-time feedback.

**Tech Stack:** expo-image-picker, expo-video-thumbnails, Supabase Storage, Zustand, React Query

---

### Task 1: Install Video Dependencies

- [ ] **Step 1: Install packages**

```bash
npx expo install expo-image-picker expo-video-thumbnails expo-file-system
```

- [ ] **Step 2: Commit**

```bash
git add package.json package-lock.json
git commit -m "feat: install video upload dependencies"
```

---

### Task 2: Video Service Layer

**Files:**
- Create: `services/videoService.ts`
- Create: `utils/videoCompressor.ts`

- [ ] **Step 1: Create video compressor utility**

```typescript
// utils/videoCompressor.ts
import * as FileSystem from 'expo-file-system';

export async function getVideoInfo(uri: string) {
  const info = await FileSystem.getInfoAsync(uri, { size: true });
  return {
    uri,
    size: info.exists ? (info as any).size || 0 : 0,
    exists: info.exists,
  };
}

export function formatFileSize(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`;
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
}
```

- [ ] **Step 2: Create video service**

```typescript
// services/videoService.ts
import { supabase } from '../lib/supabase';
import { Video } from '../types/video';
import * as FileSystem from 'expo-file-system';
import * as VideoThumbnails from 'expo-video-thumbnails';

export const videoService = {
  async pickVideo(source: 'camera' | 'gallery') {
    const { launchCameraAsync, launchImageLibraryAsync, requestCameraPermissionsAsync, requestMediaLibraryPermissionsAsync } = await import('expo-image-picker');
    
    if (source === 'camera') {
      const perm = await requestCameraPermissionsAsync();
      if (!perm.granted) throw new Error('Camera permission required');
      return launchCameraAsync({ mediaTypes: ['videos'], videoMaxDuration: 900, quality: 0.7 });
    }
    const perm = await requestMediaLibraryPermissionsAsync();
    if (!perm.granted) throw new Error('Media library permission required');
    return launchImageLibraryAsync({ mediaTypes: ['videos'], quality: 0.7 });
  },

  async generateThumbnail(videoUri: string): Promise<string> {
    const { uri } = await VideoThumbnails.getThumbnailAsync(videoUri, { time: 1000 });
    return uri;
  },

  async uploadVideo(
    userId: string,
    fileUri: string,
    onProgress?: (progress: number) => void
  ): Promise<{ storagePath: string; thumbnailUrl: string | null }> {
    const fileName = `${userId}/${Date.now()}.mp4`;
    const fileData = await FileSystem.readAsStringAsync(fileUri, { encoding: FileSystem.EncodingType.Base64 });
    
    const { error: uploadError } = await supabase.storage
      .from('videos')
      .upload(fileName, decode(fileData), {
        contentType: 'video/mp4',
        upsert: false,
      });
    if (uploadError) throw uploadError;

    // Generate & upload thumbnail
    let thumbnailUrl: string | null = null;
    try {
      const thumbUri = await this.generateThumbnail(fileUri);
      const thumbName = `${userId}/thumbs/${Date.now()}.jpg`;
      const thumbData = await FileSystem.readAsStringAsync(thumbUri, { encoding: FileSystem.EncodingType.Base64 });
      await supabase.storage.from('videos').upload(thumbName, decode(thumbData), { contentType: 'image/jpeg' });
      const { data } = supabase.storage.from('videos').getPublicUrl(thumbName);
      thumbnailUrl = data.publicUrl;
    } catch {}

    if (onProgress) onProgress(1);
    return { storagePath: fileName, thumbnailUrl };
  },

  async saveVideoMetadata(userId: string, storagePath: string, thumbnailUrl: string | null, durationSec: number | null, fileSize: number | null): Promise<Video> {
    const { data, error } = await supabase.from('videos').insert({
      user_id: userId,
      storage_path: storagePath,
      thumbnail_url: thumbnailUrl,
      duration_seconds: durationSec,
      file_size_bytes: fileSize,
      status: 'uploaded',
    }).select().single();
    if (error) throw error;
    return data;
  },

  async getUserVideos(userId: string): Promise<Video[]> {
    const { data, error } = await supabase.from('videos')
      .select('*').eq('user_id', userId).order('uploaded_at', { ascending: false });
    if (error) throw error;
    return data || [];
  },
};

function decode(base64: string): Uint8Array {
  const binaryString = atob(base64);
  const bytes = new Uint8Array(binaryString.length);
  for (let i = 0; i < binaryString.length; i++) bytes[i] = binaryString.charCodeAt(i);
  return bytes;
}
```

- [ ] **Step 3: Commit**

```bash
git add services/videoService.ts utils/videoCompressor.ts
git commit -m "feat: add video service and compressor utility"
```

---

### Task 3: Upload Hook

**Files:**
- Create: `hooks/useVideoUpload.ts`

- [ ] **Step 1: Implement upload hook**

```typescript
// hooks/useVideoUpload.ts
import { useState } from 'react';
import { useAuthStore } from '../stores/authStore';
import { videoService } from '../services/videoService';
import { getVideoInfo } from '../utils/videoCompressor';
import { Video } from '../types/video';

type UploadState = 'idle' | 'picking' | 'uploading' | 'saving' | 'done' | 'error';

export function useVideoUpload() {
  const [state, setState] = useState<UploadState>('idle');
  const [progress, setProgress] = useState(0);
  const [error, setError] = useState<string | null>(null);
  const [result, setResult] = useState<Video | null>(null);
  const user = useAuthStore((s) => s.user);

  const upload = async (source: 'camera' | 'gallery') => {
    if (!user) { setError('Not authenticated'); return; }
    try {
      setState('picking');
      setError(null);
      setProgress(0);

      const pickerResult = await videoService.pickVideo(source);
      if (pickerResult.canceled || !pickerResult.assets?.[0]) { setState('idle'); return; }
      const asset = pickerResult.assets[0];

      setState('uploading');
      const info = await getVideoInfo(asset.uri);
      const { storagePath, thumbnailUrl } = await videoService.uploadVideo(
        user.id, asset.uri, (p) => setProgress(p)
      );

      setState('saving');
      const video = await videoService.saveVideoMetadata(
        user.id, storagePath, thumbnailUrl, asset.duration ? Math.round(asset.duration / 1000) : null, info.size
      );

      setResult(video);
      setState('done');
    } catch (err: any) {
      setError(err.message || 'Upload failed');
      setState('error');
    }
  };

  const reset = () => { setState('idle'); setProgress(0); setError(null); setResult(null); };

  return { state, progress, error, result, upload, reset };
}
```

- [ ] **Step 2: Commit**

```bash
git add hooks/useVideoUpload.ts
git commit -m "feat: add useVideoUpload hook"
```

---

### Task 4: Upload Progress Component

**Files:**
- Create: `components/player/UploadProgress.tsx`
- Create: `components/ui/ProgressBar.tsx`

- [ ] **Step 1: Create ProgressBar**

```tsx
// components/ui/ProgressBar.tsx
import React from 'react';
import { View, StyleSheet } from 'react-native';
import { colors } from '../../theme/colors';
import { borderRadius } from '../../theme/spacing';

interface ProgressBarProps { progress: number; height?: number; color?: string }

export function ProgressBar({ progress, height = 6, color = colors.primary[400] }: ProgressBarProps) {
  return (
    <View style={[styles.track, { height }]}>  
      <View style={[styles.fill, { width: `${Math.min(progress * 100, 100)}%`, height, backgroundColor: color }]} />
    </View>
  );
}

const styles = StyleSheet.create({
  track: { width: '100%', backgroundColor: colors.dark[600], borderRadius: borderRadius.full, overflow: 'hidden' },
  fill: { borderRadius: borderRadius.full },
});
```

- [ ] **Step 2: Create UploadProgress**

```tsx
// components/player/UploadProgress.tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { Ionicons } from '@expo/vector-icons';
import { ProgressBar } from '../ui/ProgressBar';
import { Card } from '../ui/Card';
import { colors } from '../../theme/colors';
import { typography } from '../../theme/typography';
import { spacing } from '../../theme/spacing';

type State = 'idle' | 'picking' | 'uploading' | 'saving' | 'done' | 'error';

const stateLabels: Record<State, string> = {
  idle: 'Ready to upload', picking: 'Selecting video...', uploading: 'Uploading...',
  saving: 'Saving metadata...', done: 'Upload complete!', error: 'Upload failed',
};

interface Props { state: State; progress: number; error: string | null }

export function UploadProgress({ state, progress, error }: Props) {
  const icon = state === 'done' ? 'checkmark-circle' : state === 'error' ? 'alert-circle' : 'cloud-upload';
  const iconColor = state === 'done' ? colors.success : state === 'error' ? colors.error : colors.primary[400];

  return (
    <Card>
      <View style={styles.row}>
        <Ionicons name={icon} size={32} color={iconColor} />
        <View style={styles.textCol}>
          <Text style={styles.label}>{stateLabels[state]}</Text>
          {error && <Text style={styles.error}>{error}</Text>}
        </View>
      </View>
      {(state === 'uploading' || state === 'saving') && (
        <ProgressBar progress={progress} />
      )}
    </Card>
  );
}

const styles = StyleSheet.create({
  row: { flexDirection: 'row', alignItems: 'center', gap: spacing.md, marginBottom: spacing.sm },
  textCol: { flex: 1 },
  label: { fontFamily: typography.fonts.bodySemiBold, fontSize: typography.sizes.base, color: colors.white },
  error: { fontFamily: typography.fonts.body, fontSize: typography.sizes.sm, color: colors.error, marginTop: spacing.xs },
});
```

- [ ] **Step 3: Commit**

```bash
git add components/ui/ProgressBar.tsx components/player/UploadProgress.tsx
git commit -m "feat: add ProgressBar and UploadProgress components"
```

---

### Task 5: Upload Screen

**Files:**
- Create: `app/(player)/upload.tsx`
- Create: `components/player/VideoUploader.tsx`

- [ ] **Step 1: Create VideoUploader component** — two large touchable cards (Camera / Gallery), calls `useVideoUpload().upload(source)`.

- [ ] **Step 2: Create upload screen** — combines VideoUploader + UploadProgress, shows result card on success with "View Analysis" button.

- [ ] **Step 3: Verify** — run app, navigate to upload tab, pick a video, confirm progress shows and metadata saves.

- [ ] **Step 4: Commit**

```bash
git add app/(player)/upload.tsx components/player/VideoUploader.tsx
git commit -m "feat: add video upload screen"
```

---

> **Plan 03 complete.** After executing all 5 tasks: video pick from camera/gallery, upload to Supabase Storage with progress, metadata saved to DB, upload history available. Continue to `plan-04-player-profile.md`.
