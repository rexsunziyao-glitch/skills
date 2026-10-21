---
name: folia-case-module
description: Use when building, extending, debugging or porting the 病例配乐 (case photo collection) module in the Folia music player (folia-major), covering per-case photo collections paired 1:1 with songs, case collections shown as playlists on the Bravais wall, the case-photo mode toggle, and the Bravais "播放时墙面缓缓漫游" slow pan / Ken Burns motion while music plays.
---

# Folia 病例模组 (Case Photo Collections)

Folia lets you make a "case collection" (病例集): a named set of photos where each photo
is paired with one song. When a song plays, its paired photo replaces the album cover.
Saved case collections appear as playlist cards in the local library and on the Bravais wall.
While music plays, the collection wall drifts slowly and the focused card gets a light Ken Burns zoom.

> **Privacy rule.** Case photos are clinical images. Never commit real photos, patient names
> or identifiable metadata to the repo, tests, fixtures, screenshots or this skill. Use
> placeholder names like `Case A`, and generated or blank images. Data stays on the user's
> machine (IndexedDB), and blob URLs are never persisted.

## 1. Data model

`src/types/casePhotoCollection.ts`
```ts
interface CasePhotoEntry { id: string; name: string; blob: Blob; song: SongResult | null }
interface CasePhotoCollection { id: string; name: string; createdAt: number; updatedAt: number; entries: CasePhotoEntry[] }
```
- Storage: Dexie table `case_photo_collections` (primary key `id`) in `src/services/appDatabase.ts`.
  It is separate from the disposable music caches, so a cache wipe never deletes case data.
  Adding the table needs a DB version bump plus a migration test (`test/unit/services/appDatabaseMigration.test.ts`).
- Runtime tag on songs (`src/types.ts`, `SongResult.casePhoto`):
  `{ collectionId, entryId, coverUrl, coverHidden? }`. The tag is **stripped before saving**
  (`saveCasePhotoCollection` and `withoutCasePhoto`), so only the original song metadata is stored.
- Library projection (`src/library/core/model/casePhotoCollections.ts`): each collection becomes a
  `LocalLibraryGroup` with `id: case-collection-<id>`, `type: 'playlist'`, `casePhotoCollectionId`,
  and `trackCount` = entries that have a song. The paired songs are never rewritten here.

See `templates/example-case-collection.json` for a fake, serialized example.

## 2. Playlist pairing rules (`src/utils/casePhotoPlaylist.ts`)
1. Accepted images: `.jpg .jpeg .png .webp`. An empty or octet-stream MIME type is allowed if the extension matches.
2. Song pool (`resolveCasePhotoSongPool(needed, queue)`):
   the current queue if it has enough unique tracks → else the local library → else queue+library mixed.
   Songs are deduplicated by `getPlaybackSongKey` and shuffled. A song is **reused only after the unique pool runs out**, and the UI then shows `casePhotoPlaylist.reusedSongs`.
3. Pairing is 1:1 in photo order (`zipCasePhotoPairs`).
4. **Each case keeps its own songs.** Re-opening a collection restores its saved songs (`selectCollection`).
   `rerollOne` swaps one photo's song only, picking a song not used by the other photos in that case.
   Prefer the existing pairs. Only re-pair when there is a conflict.
5. Each entry carries its own image, so editing one case can never change another case's cover.
   `findPairForSong` matches on both `collectionId` and `entryId`.
6. Switching or creating a collection auto-saves a dirty one first.

## 3. Files map
| Area | File |
|---|---|
| Types | `src/types/casePhotoCollection.ts`, `SongResult.casePhoto` in `src/types.ts` |
| Persistence | `src/services/casePhotoCollectionService.ts`, `appDatabase.ts` |
| Runtime (blob URLs, song tagging) | `src/services/casePhotoCollectionRuntime.ts` |
| Editor state | `src/stores/useCasePhotoPlaylistStore.ts` |
| Mode on/off | `src/stores/useCasePhotoModeStore.ts`, `src/services/casePhotoModeActions.ts` (`exitCasePhotoMode`, `restoreMusicArtwork`) |
| Cover restore after relaunch | `src/hooks/useCasePhotoPlaylistVisualSync.ts`, `useSessionRestoreController.ts`, `restorePlaybackSource.ts` |
| Command palette UI | `commands/casePhotoPlaylistCommand.ts`, `casePhotoExitCommand.ts`, `surfaces/CasePhoto*` |
| Exit button overlay | `src/components/app/overlays/CasePhotoModeExitButton.tsx` |
| Library/wall cards | `library/core/model/casePhotoCollections.ts`, `bindings/useCasePhotoCollections.ts`, `suites/bravais/BravaisHomeNativeLibrary.tsx`, `BravaisHomeSources.tsx` |
| i18n | `casePhotoPlaylist.*` in `src/i18n/locales/{zh-CN,en,in}.ts` |
| Tests | `test/unit/case-photo/*`, `test/unit/library/core/casePhotoCollections.test.ts`, `test/unit/services/casePhotoCollectionService.test.ts`, `test/unit/playback/casePhotoSessionRestore.test.ts`, `test/component/casePhoto*.spec.ts` |

## 4. Adding a new case feature or module
1. Extend the types first. Keep `blob` and the original `song`, and never persist blob URLs or the `casePhoto` tag.
2. If the shape changes, bump the Dexie version and add a migration plus its test.
3. Keep editor logic in `useCasePhotoPlaylistStore`, and set `dirty: true` on every mutation.
4. Surface it through the command palette (`createToggleCommand` or a surface) and add i18n keys in all 3 locales.
   The command registry snapshot must be updated.
5. To make it show on the wall or in the library, go through `buildCasePhotoLibraryGroups`. Don't special-case suites.
6. Hiding the mode must hide patient artwork everywhere (current song, queue, transition display, cached cover)
   without deleting the data. Follow `restoreMusicArtwork`.

## 5. Bravais slow pan / Ken Burns (`feat/bravais-roam-while-playing`, commit 352c382)
- Pure rules: `src/library/suites/bravais/bravaisRoam.ts`. Hook: `useBravaisRoam.ts`. Store: `src/stores/useBravaisRoamStore.ts`.
- Setting: 界面设置 → Bravais 墙面 → 「播放时墙面缓缓漫游」 (`BravaisSettingsSection.tsx`). It is **on by default**.
  It is saved in localStorage under the key `bravais_roam_while_playing` and is also in the command palette.
  Deliberately **excluded** from the appearance shortcode, JSON import/export and the sync snapshot, the same exception as wall transparency.
- Constants (`BRAVAIS_ROAM`):
  - `panPerMinute: 0.45` of the viewport
  - Lissajous `omegaX = 2π/90`, `omegaY = 2π/70`
  - `coastMs: 800` on pause
  - `interruptResumeMs: 2000` after drag/wheel
  - the camera follows an auto song change for at most `1.2s` and at most `0.28` of the viewport
- It runs only on the `collection` surface (including case collections), and only when the preference is on, reduced-motion is off,
  music is playing, and the stage is active. Phases: `idle | roaming | coasting | interrupted`. Elapsed time freezes
  during interrupts, so the motion never jumps.
- Ken Burns is a micro zoom on the expanded focus card only (`bravais.css`, `useBravaisPlayingCard.ts`).

## 6. Verification
```bash
npm run typecheck
npm test -- casePhoto case-photo bravaisRoam bravaisPlayingCard appDatabaseMigration commandRegistry
npm run test:component -- casePhoto
```
Manual check, with placeholder images only:
1. Create a case and import 3 images. Pair them, then reroll one photo and confirm the others keep their songs.
2. Save it, then reload the app. The collection card appears and the covers restore.
3. Play from the case collection on the Bravais wall. The wall drifts visibly, coasts on pause, and dragging interrupts it for about 2s.
4. Turn the setting off and the motion stops. Turn on OS reduced-motion and the motion stops.
5. Exit case mode. All patient artwork disappears from the queue and now-playing view, and the data remains.

## 7. Status notes (as of 2026-10-21)
- The roam feature is committed on `feat/bravais-roam-while-playing` (352c382, then tuned to 0.45/min in the working tree).
- Most of the case module is still **uncommitted** working-tree changes in `~/Documents/folia-major`.
  Commit it before relying on a branch. There is also an untracked local backup dir, `.folia-case-native-backups/`. Never commit it, because it may hold real data.
- Related skill: `folia-settings-integration` (the general settings rules; roam is a documented exception).
