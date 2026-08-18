# Shorts Autopost — Multi-Platform Short-Video Publishing App

Flutter + SQLite scaffold implementing the full architecture: BYOK multi-AI
metadata generation, YouTube Shorts / Instagram Reels / Facebook Reels
publishing, 9:16 enforcement, cover-frame extraction, a hash-based dedupe
queue, and a WorkManager-driven burst scheduler.

## Project layout

```
lib/
  core/
    constants.dart          # AiProvider enum, QueueStatus, SecureKeys, tunables
    theme.dart
  database/
    app_database.dart       # sqflite schema + VideoQueueDao (dedupe-safe CRUD)
    models/video_queue_item.dart
  services/
    secure_storage_service.dart   # flutter_secure_storage wrapper (Keychain / EncryptedSharedPreferences)
    ai/
      ai_provider.dart              # shared interface + prompt template + JSON parser
      gemini_service.dart
      openai_compatible_service.dart # OpenAI, DeepSeek, and Custom-base-URL providers
      claude_service.dart
      ai_service_factory.dart       # resolves selected provider -> concrete service
    social/
      youtube_service.dart    # Data API v3 resumable upload -> Short
      meta_service.dart       # FB Page Reels (rupload) + IG Reels (container flow)
    video/
      video_processor_service.dart  # ffprobe, 9:16 blur-pad, sharpest-frame extraction
      video_import_service.dart     # bulk picker + sha256 hash dedupe
    scheduler/
      upload_pipeline.dart    # orchestrates one item: AI -> process -> publish x3 -> status
      burst_scheduler.dart    # WorkManager periodic task wrapper
  screens/
    dashboard_screen.dart     # counters, Start Upload, live logs, scheduler toggle
    queue_screen.dart         # bulk import + per-item cards
    settings_screen.dart      # AI provider dropdown, all credential fields, test buttons
  widgets/
    queue_item_card.dart
  main.dart
android/app/src/main/AndroidManifest.xml   # background/network/media permissions
```

## Setup

```bash
flutter pub get
flutter run
```

You will need platform-specific setup beyond this scaffold:
- **iOS**: add `NSPhotoLibraryUsageDescription` to Info.plist, enable Background Modes
  (Background fetch / Background processing) for the scheduler, and configure
  Keychain sharing if needed.
- **Android**: `minSdkVersion 21+`, and if targeting Android 13+, use the
  granular media permissions already declared in the manifest.
- **YouTube**: register an OAuth 2.0 client (Android/iOS type) in Google Cloud
  Console with the YouTube Data API v3 enabled; this scaffold's Settings
  screen accepts a pre-obtained access token directly (swap in a full
  `google_sign_in` + `oauth2` authorization-code flow for production —
  the `youtube_service.dart` refresh-token plumbing is stubbed for this).
- **Meta**: create a Meta App with the Pages API + Instagram Graph API
  products, generate a long-lived Page Access Token, and note the Page ID
  and linked Instagram Business Account ID.

## Key design decisions

- **Duplicate prevention** is enforced twice: a `UNIQUE` index on
  `file_hash` at the DB layer (belt), and an explicit
  `hasCompletedHash()` check before import (suspenders) — so a completed
  video can never re-enter the queue even under a different filename.
- **One AI call per video** returns metadata for all three platforms at
  once (see `buildMetadataPrompt`), keeping BYOK token spend down instead
  of firing three separate prompts.
- **Instagram Reels publishing requires a public HTTPS video URL** (Meta
  Graph API constraint) — unlike Facebook's Reels endpoint, which accepts
  raw byte upload directly. `upload_pipeline.dart` flags this explicitly;
  wire `publishInstagramReel` to your own CDN/storage uploader (or a
  signed Google Drive link) to complete that leg in production.
- **Burst scheduler** runs one queue item per WorkManager tick; the
  configured interval (e.g. every 3–4h) naturally spaces posts out.
  Android's periodic-task floor is 15 minutes — enforced in
  `BurstScheduler.enable()`.
- **9:16 enforcement** re-encodes only when needed (skips already-vertical
  sources) using an ffmpeg `split` + `boxblur` + `overlay` filter graph for
  the blurred-padding look.

## Not included in this scaffold (flagged as follow-up work)

- Full OAuth2 authorization-code UI flow for YouTube (currently accepts a
  pre-obtained token) and its refresh-token renewal logic.
- Google Drive folder-watch polling job (the DB/pipeline is ready for it —
  add a `drive_watch` background task that calls the same
  `VideoImportService` insert path).
- A CDN/storage uploader for the Instagram public-URL requirement.
- Retry/backoff policy for partial platform failures (currently marks the
  item COMPLETED if *any* platform succeeds, FAILED only if all three fail
  — tune to your preferred policy).
