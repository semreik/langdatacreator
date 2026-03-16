# Bulk Media Upload & Import Design

## Goal
Add image and audio bulk support to both BulkUpload tab (direct GitHub push) and Import tab (local staging with auto-match).

## Architecture
- Extend existing `BulkUploadTabContent` to detect media files (not just JSON) and route them to correct GitHub paths with naming instructions
- Extend existing `ImportTabContent` to accept media files, auto-match filenames to existing local cards/entries, and stage them via ImageStagingContext/AudioStagingContext
- Both tabs share the same category selector + instruction panel pattern
- PNG only for images, `.wav/.mp3/.m4a/.aac` for audio

## Media Categories & Naming Conventions

| Category | Format | Naming Convention | Target Path |
|----------|--------|-------------------|-------------|
| Deck Images | `.png` only | `{card-id}.png` | `assets/images/decks/{deckId}/{filename}` |
| Culture Images | `.png` only | `{step-id}.png` | `assets/images/culture/{filename}` |
| Dictionary Audio | `.wav .mp3 .m4a .aac` | `{word}.wav` | `assets/audio/dictionary_words/{filename}` |
| Conversation Audio | `.wav .mp3 .m4a .aac` | `{convId}_{N}_{A\|B}.wav` | `assets/audio/conversations/{convId}/{filename}` |

## UI Layout (shared by both tabs)

1. **Category selector** — buttons: Deck Images, Culture Images, Dictionary Audio, Conversation Audio
   - Deck Images shows a dropdown to select which deck
2. **Instruction box** — naming convention, format, examples, target path (changes per category)
3. **Drop zone** — drag-and-drop accepting multiple files
4. **File list** — shows dropped files with status per file

## BulkUpload Tab (Direct to GitHub)

- Format validation only (is it `.png`? is it audio format?)
- No matching against existing data
- User follows naming instructions
- Files pushed directly to correct GitHub paths
- Show upload progress per file
- Existing JSON bulk upload unchanged

## Import Tab (Local Staging)

- Same category selector and instructions
- Auto-match filenames to existing local data:
  - **Deck images:** match `{filename without ext}` to card IDs in selected deck
  - **Culture images:** stage directly (no matching needed)
  - **Dictionary audio:** match `{filename without ext}` to dictionary entry `dz` fields
  - **Conversation audio:** parse `{convId}_{N}_{A|B}` pattern, match to existing conversations
- Matched files staged into ImageStagingContext / AudioStagingContext
- Unmatched files shown with warning, not staged
- Existing JSON import unchanged

## No Changes To
- ImageStagingContext / AudioStagingContext interfaces (already support multiple files)
- Existing JSON workflows in both tabs
- Other upload tabs (Decks, Dictionary, Conversations, Culture, Files)
