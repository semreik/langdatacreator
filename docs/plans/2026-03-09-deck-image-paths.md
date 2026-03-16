# Fix Deck Image Upload Paths

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Deck images should upload to `assets/images/decks/{deckId}/{filename}` instead of `assets/images/decks/{filename}`, matching the skeleton app's expected structure.

**Architecture:** Add an optional `deckId` field to the `StagedImage` interface in `ImageStagingContext`. Thread it from `DeckView` through `ImageSection` into `stageImage()`. At upload time (both per-tab and upload-all), build the path using the deck ID subfolder.

**Tech Stack:** React, TypeScript, Next.js

---

### Task 1: Add `deckId` to ImageStagingContext

**Files:**
- Modify: `app/contexts/ImageStagingContext.tsx`

**Step 1: Add `deckId` to `StagedImage` interface**

In `app/contexts/ImageStagingContext.tsx`, change the `StagedImage` interface from:

```typescript
export interface StagedImage {
  id: string;
  cardId: string;
  cardDisplay: string;
  originalFilename: string;
  targetFilename: string;
  blob: Blob;
  url: string;
}
```

to:

```typescript
export interface StagedImage {
  id: string;
  cardId: string;
  cardDisplay: string;
  deckId?: string;             // Parent deck ID (for nested folder upload)
  originalFilename: string;
  targetFilename: string;
  blob: Blob;
  url: string;
}
```

**Step 2: Add `deckId` parameter to `stageImage` in the context interface**

Change the `stageImage` signature in `ImageStagingContextValue` from:

```typescript
  stageImage: (
    cardId: string,
    cardDisplay: string,
    blob: Blob,
    originalFilename: string
  ) => string;
```

to:

```typescript
  stageImage: (
    cardId: string,
    cardDisplay: string,
    blob: Blob,
    originalFilename: string,
    deckId?: string
  ) => string;
```

**Step 3: Update `stageImage` implementation in `ImageStagingProvider`**

Change the `stageImage` callback from:

```typescript
  const stageImage = useCallback((
    cardId: string,
    cardDisplay: string,
    blob: Blob,
    originalFilename: string
  ): string => {
    const id = `img-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    const targetFilename = generateImageFilename(cardId);
    const url = URL.createObjectURL(blob);

    const staged: StagedImage = {
      id,
      cardId,
      cardDisplay,
      originalFilename,
      targetFilename,
      blob,
      url,
    };
```

to:

```typescript
  const stageImage = useCallback((
    cardId: string,
    cardDisplay: string,
    blob: Blob,
    originalFilename: string,
    deckId?: string
  ): string => {
    const id = `img-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    const targetFilename = generateImageFilename(cardId);
    const url = URL.createObjectURL(blob);

    const staged: StagedImage = {
      id,
      cardId,
      cardDisplay,
      deckId,
      originalFilename,
      targetFilename,
      blob,
      url,
    };
```

**Step 4: Verify the app compiles**

Run: `cd /mnt/c/Users/oyle/Documents/final_final_dzardzongke/langdatacreator-main && npx next build --no-lint 2>&1 | tail -5`

Or just check TypeScript: `npx tsc --noEmit 2>&1 | head -20`

**Step 5: Commit**

```bash
git add app/contexts/ImageStagingContext.tsx
git commit -m "feat: add deckId to StagedImage interface"
```

---

### Task 2: Thread `deckId` through ImageSection

**Files:**
- Modify: `app/components/ImageSection.tsx`

**Step 1: Add `deckId` to props interface**

Change:

```typescript
interface ImageSectionProps {
  cardId: string;
  cardDisplay: string;
  disabled?: boolean;
}
```

to:

```typescript
interface ImageSectionProps {
  cardId: string;
  cardDisplay: string;
  deckId?: string;
  disabled?: boolean;
}
```

**Step 2: Accept and pass `deckId` in the component**

Change the component destructuring from:

```typescript
export default function ImageSection({
  cardId,
  cardDisplay,
  disabled = false,
}: ImageSectionProps) {
```

to:

```typescript
export default function ImageSection({
  cardId,
  cardDisplay,
  deckId,
  disabled = false,
}: ImageSectionProps) {
```

**Step 3: Pass `deckId` to `stageImage` call**

Change line 45 from:

```typescript
    stageImage(cardId, cardDisplay, file, file.name);
```

to:

```typescript
    stageImage(cardId, cardDisplay, file, file.name, deckId);
```

**Step 4: Commit**

```bash
git add app/components/ImageSection.tsx
git commit -m "feat: thread deckId through ImageSection component"
```

---

### Task 3: Pass `currentDeck.id` from DeckView

**Files:**
- Modify: `app/components/DeckView.tsx`

**Step 1: Add `deckId` prop to ImageSection usage**

At line 565-568, change:

```tsx
          <ImageSection
            cardId={cardForm.id}
            cardDisplay={cardForm.front || cardForm.id}
          />
```

to:

```tsx
          <ImageSection
            cardId={cardForm.id}
            cardDisplay={cardForm.front || cardForm.id}
            deckId={currentDeck.id}
          />
```

**Step 2: Commit**

```bash
git add app/components/DeckView.tsx
git commit -m "feat: pass deck ID to ImageSection for nested upload paths"
```

---

### Task 4: Fix upload path in DecksTabContent

**Files:**
- Modify: `app/components/github-popup/DecksTabContent.tsx`

**Step 1: Update the upload path**

At line 107, change:

```typescript
        const result = await uploadBinaryFile(config as GitHubConfig, new File([img.blob], img.targetFilename, { type: img.blob.type }), `assets/images/decks/${img.targetFilename}`, `Add image`);
```

to:

```typescript
        const deckFolder = img.deckId ? `${img.deckId}/` : '';
        const result = await uploadBinaryFile(config as GitHubConfig, new File([img.blob], img.targetFilename, { type: img.blob.type }), `assets/images/decks/${deckFolder}${img.targetFilename}`, `Add image`);
```

**Step 2: Commit**

```bash
git add app/components/github-popup/DecksTabContent.tsx
git commit -m "fix: nest deck images under deck ID subfolder"
```

---

### Task 5: Fix upload path in UploadAllTabContent

**Files:**
- Modify: `app/components/github-popup/UploadAllTabContent.tsx`

**Step 1: Update the path check (line 407)**

Change:

```typescript
      const deckImgPaths = deckImages.map(i => `assets/images/decks/${i.targetFilename}`);
```

to:

```typescript
      const deckImgPaths = deckImages.map(i => {
        const deckFolder = i.deckId ? `${i.deckId}/` : '';
        return `assets/images/decks/${deckFolder}${i.targetFilename}`;
      });
```

**Step 2: Update the upload call (line 413)**

Change:

```typescript
            const r = await uploadBinaryFile(cfg, new File([img.blob], img.targetFilename, { type: img.blob.type }), `assets/images/decks/${img.targetFilename}`, `Add image`);
```

to:

```typescript
            const deckFolder = img.deckId ? `${img.deckId}/` : '';
            const r = await uploadBinaryFile(cfg, new File([img.blob], img.targetFilename, { type: img.blob.type }), `assets/images/decks/${deckFolder}${img.targetFilename}`, `Add image`);
```

**Step 3: Verify the app compiles**

Run: `npx tsc --noEmit 2>&1 | head -20`

**Step 4: Commit**

```bash
git add app/components/github-popup/UploadAllTabContent.tsx
git commit -m "fix: nest deck images under deck ID subfolder in Upload All"
```

---

### Task 6: Manual verification

**Step 1: Run dev server and test**

Run: `npm run dev`

1. Create a deck with ID `animals`
2. Add a card with an image
3. Open GitHub Upload popup > Decks > Images tab
4. Verify the upload path shown is `assets/images/decks/animals/{filename}.png`

**Step 2: Final commit (if any cleanup needed)**
