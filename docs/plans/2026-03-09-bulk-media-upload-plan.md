# Bulk Media Upload & Import Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add image and audio bulk support to the BulkUpload tab (direct GitHub push) and Import tab (local staging with auto-match to existing cards/entries).

**Architecture:** Extend `BulkUploadTabContent` and `ImportTabContent` to accept media files alongside JSON. Both tabs get a shared media category selector and instruction panel component. BulkUpload does format validation + direct GitHub push. Import does filename-to-card matching + staging via existing contexts.

**Tech Stack:** React, TypeScript, Next.js, GitHub API

---

### Task 1: Create shared MediaInstructions component

This component renders the category selector buttons, deck dropdown (when applicable), and naming convention instructions. Shared between BulkUpload and Import tabs.

**Files:**
- Create: `app/components/github-popup/MediaInstructions.tsx`

**Step 1: Create the component**

```tsx
'use client';

import React from 'react';

export type MediaCategory = 'deck-images' | 'culture-images' | 'dict-audio' | 'conv-audio';

interface MediaInstructionsProps {
  category: MediaCategory;
  onCategoryChange: (cat: MediaCategory) => void;
  selectedDeckId?: string;
  onDeckChange?: (deckId: string) => void;
  decks?: Array<{ id: string; title: string }>;
}

const categories: Array<{ value: MediaCategory; label: string; icon: string }> = [
  {
    value: 'deck-images',
    label: 'Deck Images',
    icon: 'M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z',
  },
  {
    value: 'culture-images',
    label: 'Culture Images',
    icon: 'M3.055 11H5a2 2 0 012 2v1a2 2 0 002 2 2 2 0 012 2v2.945M8 3.935V5.5A2.5 2.5 0 0010.5 8h.5a2 2 0 012 2 2 2 0 104 0 2 2 0 012-2h1.064M15 20.488V18a2 2 0 012-2h3.064M21 12a9 9 0 11-18 0 9 9 0 0118 0z',
  },
  {
    value: 'dict-audio',
    label: 'Dict Audio',
    icon: 'M19 11a7 7 0 01-7 7m0 0a7 7 0 01-7-7m7 7v4m0 0H8m4 0h4m-4-8a3 3 0 01-3-3V5a3 3 0 116 0v6a3 3 0 01-3 3z',
  },
  {
    value: 'conv-audio',
    label: 'Conv Audio',
    icon: 'M8 12h.01M12 12h.01M16 12h.01M21 12c0 4.418-4.03 8-9 8a9.863 9.863 0 01-4.255-.949L3 20l1.395-3.72C3.512 15.042 3 13.574 3 12c0-4.418 4.03-8 9-8s9 3.582 9 8z',
  },
];

const instructions: Record<MediaCategory, { format: string; naming: string; example: string; targetPath: string }> = {
  'deck-images': {
    format: 'PNG only (.png)',
    naming: 'Name each file to match the card ID in the deck',
    example: 'bird.png, cat.png, dog.png',
    targetPath: 'assets/images/decks/{deck-id}/',
  },
  'culture-images': {
    format: 'PNG only (.png)',
    naming: 'Name each file to match the image step src field',
    example: 'culture1.png, map-overview.png',
    targetPath: 'assets/images/culture/',
  },
  'dict-audio': {
    format: 'Audio (.wav, .mp3, .m4a, .aac)',
    naming: 'Name each file to match the Dzongkha word',
    example: 'hello.wav, water.wav, bird.mp3',
    targetPath: 'assets/audio/dictionary_words/',
  },
  'conv-audio': {
    format: 'Audio (.wav, .mp3, .m4a, .aac)',
    naming: 'Name: {conversationId}_{exchangeNumber}_{A or B}',
    example: 'greetings_1_A.wav, greetings_2_B.wav',
    targetPath: 'assets/audio/conversations/{conversationId}/',
  },
};

export default function MediaInstructions({
  category,
  onCategoryChange,
  selectedDeckId,
  onDeckChange,
  decks,
}: MediaInstructionsProps) {
  const info = instructions[category];

  return (
    <div className="space-y-3">
      {/* Category selector */}
      <div>
        <label className="block text-xs font-medium text-[var(--color-text-muted)] uppercase tracking-wide mb-2">
          Media Type
        </label>
        <div className="grid grid-cols-2 gap-2">
          {categories.map((cat) => (
            <button
              key={cat.value}
              onClick={() => onCategoryChange(cat.value)}
              className={`
                flex items-center gap-2 p-2 rounded-lg text-xs transition-all
                ${category === cat.value
                  ? 'bg-[var(--color-primary)] text-white'
                  : 'bg-[var(--color-bg)] text-[var(--color-text-muted)] hover:bg-[var(--color-hover-overlay)] hover:text-[var(--color-text)]'
                }
              `}
            >
              <svg className="w-4 h-4 flex-shrink-0" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d={cat.icon} />
              </svg>
              <span className="truncate">{cat.label}</span>
            </button>
          ))}
        </div>
      </div>

      {/* Deck selector (only for deck images) */}
      {category === 'deck-images' && decks && onDeckChange && (
        <div>
          <label className="block text-xs font-medium text-[var(--color-text-muted)] uppercase tracking-wide mb-1">
            Target Deck
          </label>
          <select
            value={selectedDeckId || ''}
            onChange={(e) => onDeckChange(e.target.value)}
            className="w-full p-2 rounded-lg text-sm bg-[var(--color-bg)] border border-[var(--color-border)] text-[var(--color-text)] outline-none focus:border-[var(--color-primary)]"
          >
            <option value="">Select a deck...</option>
            {decks.map((d) => (
              <option key={d.id} value={d.id}>{d.title} ({d.id})</option>
            ))}
          </select>
        </div>
      )}

      {/* Instruction box */}
      <div className="p-3 rounded-lg bg-[var(--color-info)]/10 border border-[var(--color-info)]/20 space-y-1.5">
        <div className="flex items-center gap-1.5">
          <svg className="w-3.5 h-3.5 text-[var(--color-info)] flex-shrink-0" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
          </svg>
          <span className="text-xs font-semibold text-[var(--color-info)]">Naming Guide</span>
        </div>
        <div className="text-xs text-[var(--color-info)] space-y-0.5">
          <p><strong>Format:</strong> {info.format}</p>
          <p><strong>Naming:</strong> {info.naming}</p>
          <p><strong>Example:</strong> <code className="bg-[var(--color-info)]/10 px-1 rounded">{info.example}</code></p>
          <p><strong>Target:</strong> <code className="bg-[var(--color-info)]/10 px-1 rounded">
            {category === 'deck-images' && selectedDeckId
              ? info.targetPath.replace('{deck-id}', selectedDeckId)
              : info.targetPath}
          </code></p>
        </div>
      </div>
    </div>
  );
}

export function getAcceptString(category: MediaCategory): string {
  if (category === 'deck-images' || category === 'culture-images') {
    return 'image/png';
  }
  return 'audio/wav,audio/mpeg,audio/mp4,audio/aac,audio/x-m4a,.wav,.mp3,.m4a,.aac';
}

export function isValidMediaFile(file: File, category: MediaCategory): boolean {
  if (category === 'deck-images' || category === 'culture-images') {
    return file.type === 'image/png' || file.name.toLowerCase().endsWith('.png');
  }
  const audioExts = ['.wav', '.mp3', '.m4a', '.aac'];
  return audioExts.some(ext => file.name.toLowerCase().endsWith(ext));
}

export function getTargetPath(file: File, category: MediaCategory, deckId?: string): string {
  switch (category) {
    case 'deck-images':
      return `assets/images/decks/${deckId || 'unknown'}/${file.name}`;
    case 'culture-images':
      return `assets/images/culture/${file.name}`;
    case 'dict-audio':
      return `assets/audio/dictionary_words/${file.name}`;
    case 'conv-audio': {
      // Parse conversationId from filename: {convId}_{N}_{A|B}.ext
      const basename = file.name.replace(/\.[^.]+$/, '');
      const match = basename.match(/^(.+?)_(\d+)_([AB])$/);
      const convId = match ? match[1] : 'unknown';
      return `assets/audio/conversations/${convId}/${file.name}`;
    }
  }
}
```

**Step 2: Verify it compiles**

Run: `npx tsc --noEmit 2>&1 | head -20`

---

### Task 2: Add media support to BulkUploadTabContent

Extend the existing BulkUpload tab to handle media files alongside JSON. When the user drops media files, show a media category selector and instructions, then upload directly to GitHub.

**Files:**
- Modify: `app/components/github-popup/BulkUploadTabContent.tsx`

**Step 1: Add media imports and state**

At the top of the file, add these imports after the existing ones:

```tsx
import { uploadBinaryFile } from '../../utils/githubApi';
import { useDecks } from '../../contexts/DecksContext';
import MediaInstructions, { MediaCategory, getAcceptString, isValidMediaFile, getTargetPath } from './MediaInstructions';
```

**Step 2: Add media state variables**

Inside the `BulkUploadTabContent` component, after the existing state variables (around line 272), add:

```tsx
  // Media upload state
  const { decks: deckMetas } = useDecks();
  const [mode, setMode] = useState<'json' | 'media'>('json');
  const [mediaCategory, setMediaCategory] = useState<MediaCategory>('deck-images');
  const [selectedDeckId, setSelectedDeckId] = useState<string>('');
  const [mediaFiles, setMediaFiles] = useState<File[]>([]);
```

**Step 3: Update `processFiles` to detect media vs JSON**

Replace the existing `processFiles` function with one that detects file types and routes accordingly. If any dropped file is NOT a JSON file, switch to media mode:

```tsx
  const processFiles = (fileList: FileList) => {
    setParseError(null);
    setParsedFiles([]);
    setMediaFiles([]);
    setSteps([]);
    setIsDone(false);

    const allFiles = Array.from(fileList);
    const jsonFiles = allFiles.filter(f =>
      f.type === 'application/json' || f.name.endsWith('.json')
    );
    const nonJsonFiles = allFiles.filter(f =>
      !f.type.startsWith('application/json') && !f.name.endsWith('.json')
    );

    // If we have non-JSON files, switch to media mode
    if (nonJsonFiles.length > 0) {
      setMode('media');
      // Auto-detect category from file types
      const hasImages = nonJsonFiles.some(f => f.name.toLowerCase().endsWith('.png'));
      const hasAudio = nonJsonFiles.some(f =>
        ['.wav', '.mp3', '.m4a', '.aac'].some(ext => f.name.toLowerCase().endsWith(ext))
      );

      if (hasImages && !hasAudio) {
        // Default to deck-images, user can change
        setMediaCategory('deck-images');
      } else if (hasAudio && !hasImages) {
        setMediaCategory('dict-audio');
      }

      // Filter to only valid files for detected category
      setMediaFiles(nonJsonFiles);
      return;
    }

    // Otherwise, JSON mode (existing logic)
    setMode('json');

    if (jsonFiles.length === 0) {
      setParseError('No supported files found. Drop JSON, PNG, or audio files.');
      return;
    }

    const results: typeof parsedFiles = [];
    let processed = 0;

    jsonFiles.forEach(file => {
      const reader = new FileReader();
      reader.onload = () => {
        processed++;
        try {
          const data = JSON.parse(reader.result as string);
          const { detected, confidence } = detectDataType(data);

          if (detected) {
            results.push({
              fileName: file.name,
              data,
              type: detected,
              confidence,
              itemCount: getItemCount(data, detected),
            });
          } else {
            results.push({
              fileName: file.name,
              data,
              type: 'dictionary',
              confidence: 'Unknown format (defaulting to Dictionary)',
              itemCount: 0,
            });
          }
        } catch (e) {
          setParseError(`Failed to parse ${file.name}: ${e instanceof Error ? e.message : 'Invalid JSON'}`);
        }

        if (processed === jsonFiles.length) {
          setParsedFiles(results);
        }
      };
      reader.readAsText(file);
    });
  };
```

**Step 4: Add media upload handler**

After the existing `handleBulkUpload` function, add:

```tsx
  const handleMediaUpload = async () => {
    if (!configured || mediaFiles.length === 0) return;
    if (mediaCategory === 'deck-images' && !selectedDeckId) {
      setParseError('Please select a target deck first.');
      return;
    }

    setIsUploading(true);
    setIsDone(false);
    setProgress(0);

    const cfg = config as GitHubConfig;
    const validFiles = mediaFiles.filter(f => isValidMediaFile(f, mediaCategory));
    const invalidFiles = mediaFiles.filter(f => !isValidMediaFile(f, mediaCategory));

    const newSteps: UploadStep[] = [
      ...validFiles.map((f, i) => ({
        id: `media-${i}`,
        label: f.name,
        status: 'pending' as const,
      })),
      ...invalidFiles.map((f, i) => ({
        id: `invalid-${i}`,
        label: f.name,
        status: 'skipped' as const,
        message: 'Wrong file format — skipped',
      })),
    ];
    setSteps(newSteps);

    let completed = 0;
    const updateStep = (id: string, update: Partial<UploadStep>) => {
      setSteps(prev => prev.map(s => s.id === id ? { ...s, ...update } : s));
    };

    for (let i = 0; i < validFiles.length; i++) {
      const file = validFiles[i];
      const stepId = `media-${i}`;
      updateStep(stepId, { status: 'uploading' });

      try {
        const targetPath = getTargetPath(file, mediaCategory, selectedDeckId);
        const r = await uploadBinaryFile(cfg, file, targetPath, `Add media: ${file.name}`);
        if (r.success) {
          updateStep(stepId, { status: 'done', message: targetPath });
        } else {
          updateStep(stepId, { status: 'error', error: r.error || 'Upload failed' });
        }
      } catch (e) {
        updateStep(stepId, { status: 'error', error: e instanceof Error ? e.message : 'Unknown error' });
      }

      completed++;
      setProgress(Math.round((completed / validFiles.length) * 100));
    }

    setIsUploading(false);
    setIsDone(true);
  };
```

**Step 5: Update the reset function**

```tsx
  const reset = () => {
    setParsedFiles([]);
    setMediaFiles([]);
    setSteps([]);
    setIsDone(false);
    setProgress(0);
    setParseError(null);
    setMode('json');
  };
```

**Step 6: Update the file input accept attribute and drop zone**

In the JSX, update the `<input>` element's accept to also accept media:

```tsx
accept=".json,application/json,image/png,.png,audio/*,.wav,.mp3,.m4a,.aac"
```

Update the drop zone text:

```tsx
<p className="text-sm font-medium text-[var(--color-text)]">
  Drop files here or click to browse
</p>
<p className="text-xs text-[var(--color-text-muted)] mt-1">
  JSON data, PNG images, or audio files — pushes directly to GitHub
</p>
```

**Step 7: Update the info banner text**

```tsx
<p className="text-xs text-[var(--color-info)]">
  Drop files to push directly to GitHub. Supports JSON data, PNG images, and audio files. Auto-detects type and merges JSON with existing data.
</p>
```

**Step 8: Add media mode UI between the drop zone and the parsed files list**

After the drop zone section (after the closing `)}` around line 559), and before the parse error section, add the media mode UI:

```tsx
{/* Media mode: show instructions + file list */}
{mode === 'media' && mediaFiles.length > 0 && !isUploading && !isDone && (
  <div className="space-y-3">
    <MediaInstructions
      category={mediaCategory}
      onCategoryChange={(cat) => {
        setMediaCategory(cat);
        setSteps([]);
      }}
      selectedDeckId={selectedDeckId}
      onDeckChange={setSelectedDeckId}
      decks={deckMetas.map(d => ({ id: d.id, title: d.title }))}
    />

    {/* File list */}
    <div className="space-y-1.5">
      <div className="flex items-center justify-between">
        <p className="text-xs font-semibold tracking-wider uppercase text-[var(--color-text-muted)]">
          {mediaFiles.length} file{mediaFiles.length > 1 ? 's' : ''}
        </p>
        <button
          onClick={reset}
          className="text-xs text-[var(--color-text-muted)] hover:text-[var(--color-text)] transition-colors"
        >
          Clear all
        </button>
      </div>
      <div className="max-h-40 overflow-y-auto space-y-1">
        {mediaFiles.map((file, i) => {
          const valid = isValidMediaFile(file, mediaCategory);
          return (
            <div key={i} className={`flex items-center gap-2 px-3 py-1.5 rounded-lg text-xs ${
              valid
                ? 'bg-[var(--color-bg-secondary)] text-[var(--color-text)]'
                : 'bg-red-500/10 text-red-500'
            }`}>
              <span className="flex-1 truncate font-mono">{file.name}</span>
              {valid ? (
                <span className="text-[var(--color-text-muted)] truncate">
                  → {getTargetPath(file, mediaCategory, selectedDeckId).split('/').slice(-2).join('/')}
                </span>
              ) : (
                <span className="text-red-400">wrong format</span>
              )}
            </div>
          );
        })}
      </div>
    </div>
  </div>
)}
```

**Step 9: Update the action button section**

Replace the existing action button conditional to handle both modes:

```tsx
{/* Action buttons */}
{((mode === 'json' && parsedFiles.length > 0) || (mode === 'media' && mediaFiles.length > 0)) && !isDone && (
  <Button
    onClick={mode === 'json' ? handleBulkUpload : handleMediaUpload}
    disabled={!configured || isUploading || (mode === 'media' && mediaCategory === 'deck-images' && !selectedDeckId)}
    isLoading={isUploading}
    fullWidth
    size="sm"
  >
    {isUploading
      ? 'Pushing to GitHub...'
      : mode === 'json'
        ? `Push ${parsedFiles.length} file${parsedFiles.length > 1 ? 's' : ''} to GitHub`
        : `Push ${mediaFiles.filter(f => isValidMediaFile(f, mediaCategory)).length} media file${mediaFiles.length > 1 ? 's' : ''} to GitHub`
    }
  </Button>
)}
```

**Step 10: Verify it compiles**

Run: `npx tsc --noEmit 2>&1 | head -20`

---

### Task 3: Add media support to ImportTabContent

Extend the Import tab to accept media files, auto-match them to existing local cards/entries, and stage them.

**Files:**
- Modify: `app/components/github-popup/ImportTabContent.tsx`

**Step 1: Add media imports**

At the top of the file, add after existing imports:

```tsx
import { useImageStaging } from '../../contexts/ImageStagingContext';
import { useAudioStaging } from '../../contexts/AudioStagingContext';
import { useDecks } from '../../contexts/DecksContext';
import { Deck, Card as FlashCard } from '../../models/Deck';
import MediaInstructions, { MediaCategory, isValidMediaFile } from './MediaInstructions';
```

**Step 2: Add media state and context hooks**

Inside the `ImportTabContent` component, after the existing state variables, add:

```tsx
  const { stageImage } = useImageStaging();
  const { stageAudio } = useAudioStaging();
  const { decks: deckMetas } = useDecks();
  const { entries } = useDictionary();
  const { categories: convCategories } = useConversations();

  const [mode, setMode] = useState<'json' | 'media'>('json');
  const [mediaCategory, setMediaCategory] = useState<MediaCategory>('deck-images');
  const [selectedDeckId, setSelectedDeckId] = useState<string>('');
  const [mediaResults, setMediaResults] = useState<Array<{
    filename: string;
    matched: boolean;
    matchedTo?: string;
    error?: string;
  }>>([]);
  const [isProcessingMedia, setIsProcessingMedia] = useState(false);
```

**Step 3: Add deck loader helper**

Add a helper to load the full deck (with cards) from localStorage, same pattern as DecksTabContent:

```tsx
const DECKS_STORAGE_KEY = 'dzardzongke-decks';

function loadDeckFromStorage(id: string): Deck | null {
  try {
    const stored = localStorage.getItem(`${DECKS_STORAGE_KEY}-deck-${id}`);
    if (!stored) return null;
    const parsed = JSON.parse(stored);
    const cards = (parsed.cards || []).map((c: Record<string, unknown>) =>
      new FlashCard(c.id as string, c.front as string, c.back as string, (c.notes as string) || '', (c.image as string) || ''));
    return new Deck(parsed.id, parsed.title, parsed.description, cards);
  } catch { return null; }
}
```

**Step 4: Add media processing function**

```tsx
  const processMediaFiles = async (files: File[]) => {
    setIsProcessingMedia(true);
    setMediaResults([]);
    const results: typeof mediaResults = [];

    for (const file of files) {
      if (!isValidMediaFile(file, mediaCategory)) {
        results.push({ filename: file.name, matched: false, error: 'Wrong format' });
        continue;
      }

      const basename = file.name.replace(/\.[^.]+$/, '');

      switch (mediaCategory) {
        case 'deck-images': {
          if (!selectedDeckId) {
            results.push({ filename: file.name, matched: false, error: 'No deck selected' });
            break;
          }
          const deck = loadDeckFromStorage(selectedDeckId);
          if (!deck) {
            results.push({ filename: file.name, matched: false, error: 'Deck not found in local data' });
            break;
          }
          const card = deck.getAllCards().find(c => c.id === basename);
          if (card) {
            stageImage(card.id, card.front || card.id, file, file.name, selectedDeckId);
            results.push({ filename: file.name, matched: true, matchedTo: `Card: ${card.front || card.id}` });
          } else {
            results.push({ filename: file.name, matched: false, error: `No card with ID "${basename}" in deck "${deck.title}"` });
          }
          break;
        }

        case 'culture-images': {
          // Culture images don't need matching — stage directly
          stageImage(`culture-${basename}`, basename, file, file.name);
          results.push({ filename: file.name, matched: true, matchedTo: 'Culture image (staged)' });
          break;
        }

        case 'dict-audio': {
          const entry = entries.find(e => e.dz === basename || e.dz.toLowerCase() === basename.toLowerCase());
          if (entry) {
            await stageAudio(entry.dz, file, file.name);
            results.push({ filename: file.name, matched: true, matchedTo: `Word: ${entry.dz} (${entry.en})` });
          } else {
            results.push({ filename: file.name, matched: false, error: `No dictionary entry with word "${basename}"` });
          }
          break;
        }

        case 'conv-audio': {
          // Parse {convId}_{N}_{A|B} pattern
          const match = basename.match(/^(.+?)_(\d+)_([AB])$/);
          if (!match) {
            results.push({ filename: file.name, matched: false, error: 'Filename must match pattern: {convId}_{number}_{A|B}' });
            break;
          }
          const [, convId, numStr, speaker] = match;
          // Check if conversation exists
          let found = false;
          for (const cat of convCategories) {
            const conv = cat.conversations?.find((c: { id: string }) => c.id === convId);
            if (conv) {
              found = true;
              break;
            }
          }
          if (found) {
            const targetPath = `assets/audio/conversations/${convId}/${file.name}`;
            await stageAudio(convId, file, file.name, targetPath);
            results.push({ filename: file.name, matched: true, matchedTo: `Conversation: ${convId}, exchange ${numStr}, speaker ${speaker}` });
          } else {
            results.push({ filename: file.name, matched: false, error: `No conversation with ID "${convId}"` });
          }
          break;
        }
      }
    }

    setMediaResults(results);
    setIsProcessingMedia(false);
  };
```

**Step 5: Update file handling to detect media**

Update the `handleDrop` function to handle both JSON and media:

```tsx
  const handleDrop = (e: React.DragEvent) => {
    e.preventDefault();
    setIsDragging(false);

    const files = Array.from(e.dataTransfer.files);
    const jsonFiles = files.filter(f => f.type === 'application/json' || f.name.endsWith('.json'));
    const mediaFiles = files.filter(f => !f.type.startsWith('application/json') && !f.name.endsWith('.json'));

    if (mediaFiles.length > 0) {
      setMode('media');
      processMediaFiles(mediaFiles);
    } else if (jsonFiles.length > 0) {
      setMode('json');
      processFile(jsonFiles[0]);
    }
  };
```

Update `handleFileSelect`:

```tsx
  const handleFileSelect = (e: React.ChangeEvent<HTMLInputElement>) => {
    const files = Array.from(e.target.files || []);
    if (files.length === 0) return;

    const jsonFiles = files.filter(f => f.type === 'application/json' || f.name.endsWith('.json'));
    const mediaFiles = files.filter(f => !f.type.startsWith('application/json') && !f.name.endsWith('.json'));

    if (mediaFiles.length > 0) {
      setMode('media');
      processMediaFiles(mediaFiles);
    } else if (jsonFiles.length > 0) {
      setMode('json');
      processFile(jsonFiles[0]);
    }

    if (fileInputRef.current) {
      fileInputRef.current.value = '';
    }
  };
```

**Step 6: Update the file input accept attribute**

Change from:
```tsx
accept=".json,application/json"
```
to:
```tsx
accept=".json,application/json,image/png,.png,audio/*,.wav,.mp3,.m4a,.aac"
```

**Step 7: Update drop zone text**

Change:
```tsx
<p className="text-sm font-medium text-[var(--color-text)]">
  Drop JSON file here or click to browse
</p>
<p className="text-xs text-[var(--color-text-muted)] mt-1">
  Supports dictionary, conversations, culture notes, and decks
</p>
```
to:
```tsx
<p className="text-sm font-medium text-[var(--color-text)]">
  Drop files here or click to browse
</p>
<p className="text-xs text-[var(--color-text-muted)] mt-1">
  JSON data, PNG images, or audio files — imports to local state
</p>
```

**Step 8: Add media mode UI and add `multiple` to the file input**

Add `multiple` attribute to the `<input>` element.

After the existing results section (before the closing `</div>` of the component), add the media UI section:

```tsx
{/* Media mode */}
{mode === 'media' && (
  <div className="space-y-3">
    <MediaInstructions
      category={mediaCategory}
      onCategoryChange={(cat) => {
        setMediaCategory(cat);
        setMediaResults([]);
      }}
      selectedDeckId={selectedDeckId}
      onDeckChange={setSelectedDeckId}
      decks={deckMetas.map(d => ({ id: d.id, title: d.title }))}
    />

    {/* Media results */}
    {mediaResults.length > 0 && (
      <div className="space-y-1.5">
        <p className="text-xs font-semibold tracking-wider uppercase text-[var(--color-text-muted)]">
          {mediaResults.filter(r => r.matched).length}/{mediaResults.length} matched & staged
        </p>
        <div className="max-h-40 overflow-y-auto space-y-1">
          {mediaResults.map((r, i) => (
            <div key={i} className={`flex items-center gap-2 px-3 py-1.5 rounded-lg text-xs ${
              r.matched
                ? 'bg-emerald-500/10 text-emerald-600 dark:text-emerald-400'
                : 'bg-red-500/10 text-red-600 dark:text-red-400'
            }`}>
              <svg className="w-3.5 h-3.5 flex-shrink-0" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                {r.matched
                  ? <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M5 13l4 4L19 7" />
                  : <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12" />
                }
              </svg>
              <span className="font-mono truncate">{r.filename}</span>
              <span className="flex-shrink-0 truncate">
                {r.matched ? `→ ${r.matchedTo}` : r.error}
              </span>
            </div>
          ))}
        </div>
      </div>
    )}

    {isProcessingMedia && (
      <div className="flex items-center justify-center gap-2 py-4">
        <div className="w-5 h-5 border-2 border-[var(--color-primary)] border-t-transparent rounded-full animate-spin" />
        <span className="text-sm text-[var(--color-text-muted)]">Matching files...</span>
      </div>
    )}

    {/* Reset button */}
    {mediaResults.length > 0 && (
      <button
        onClick={() => { setMode('json'); setMediaResults([]); }}
        className="w-full text-xs text-center text-[var(--color-text-muted)] hover:text-[var(--color-text)] py-2 transition-colors"
      >
        Back to JSON import
      </button>
    )}
  </div>
)}
```

**Step 9: Verify it compiles**

Run: `npx tsc --noEmit 2>&1 | head -30`

Note: The `stageAudio` function signature may need checking. Look at `AudioStagingContext.tsx` to verify the exact parameters it expects and adjust the call accordingly. The key parameters are: `wordId` (string), `blob` (Blob/File), `originalFilename` (string), and optionally `targetPath` (string).

---

### Task 4: Verify and fix AudioStagingContext integration

The Import tab needs to call `stageAudio()` correctly. Check the actual signature and fix any mismatches.

**Files:**
- Check: `app/contexts/AudioStagingContext.tsx` — verify `stageAudio` signature
- Possibly modify: `app/components/github-popup/ImportTabContent.tsx` — fix calls if needed

**Step 1: Read the `stageAudio` interface and implementation**

Check `AudioStagingContext.tsx` for the exact `stageAudio` signature. Key things to verify:
- What parameters does it take?
- Does it accept a `File` or only a `Blob`?
- Does it accept an optional `targetPath`?

**Step 2: Fix `processMediaFiles` calls to match**

If the signature differs from what Task 3 assumed, update the `stageAudio` calls in `processMediaFiles`. Common adjustments might be:
- Wrapping `File` as `Blob`
- Adjusting parameter order
- Adding/removing optional parameters

**Step 3: Verify it compiles**

Run: `npx tsc --noEmit 2>&1 | head -30`

---

### Task 5: Test the complete flow

**Step 1: Run the dev server**

Run: `npm run dev`

**Step 2: Test BulkUpload media flow**

1. Open the app → GitHub Upload popup → Bulk Upload tab
2. Drop a `.png` file — should switch to media mode
3. Verify category selector and instruction panel appear
4. Select "Deck Images" → verify deck dropdown appears
5. Select a deck → verify target paths update in file list
6. Drop an `.mp3` file — should auto-detect audio category

**Step 3: Test Import media flow**

1. Switch to Import tab
2. Select "Deck Images" media category
3. Select a deck that has cards
4. Drop PNG files named to match card IDs
5. Verify matched files show green with card name
6. Verify unmatched files show red with error message
7. Check that matched images appear in the staging system (Decks tab → Images section)

**Step 4: Test JSON still works**

1. Drop a JSON file in BulkUpload → should still work as before
2. Drop a JSON file in Import → should still work as before
