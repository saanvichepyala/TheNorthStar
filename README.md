# North Star

*"The internet rewards attention. We reward listening."*

A daily empathy ritual, not a social network: one shared question a day,
a single 3-minute recording, and a 6-minute listening goal before your
streak counts. No likes, no followers, no feed.

## Stack

- **Frontend:** Flutter (Clean Architecture: `domain/` → `data/` → `presentation/`)
- **Backend:** Firebase (Auth, Firestore, Storage, Cloud Messaging)
- **State:** `provider`

## Project layout

```
lib/
  core/               theming, constants, services, shared widgets, AppState
  domain/              entities + repository interfaces (no Firebase types)
  data/                 Firestore/Storage-backed repository implementations
  presentation/    onboarding, auth, home, recording, listening, profile
```

## Quick preview (no Firebase project needed)

There's a second entry point, `lib/main_mock.dart`, wired to in-memory
mock repositories (`lib/data/mock/`) instead of Firestore. It skips
auth entirely — onboarding's "Begin Listening" goes straight to Home —
and Listening Mode plays public sample videos so there's something to
actually listen to. Use this to see the app running in minutes:

```bash
flutter pub get
flutter run -t lib/main_mock.dart
```

Notes:
- The recorder still tries to use a real camera. On a simulator with
  no camera it falls back to a "Simulate a Recording" button so you
  can still walk through the full daily ritual.
- None of this data persists between runs — it's an in-memory preview,
  not a substitute for the real backend below.

## Getting it running

This repo was generated without a Flutter SDK available in the build
environment, so it hasn't been compiled or run here — you'll need to do
that locally. Steps:

1. **Install Flutter** (3.22+) — https://docs.flutter.dev/get-started/install
2. **Install dependencies**
   ```bash
   flutter pub get
   ```
3. **Create a Firebase project** at https://console.firebase.google.com,
   enabling Authentication (Google, Apple, Email/Password), Firestore,
   Storage, and Cloud Messaging.
4. **Generate `lib/firebase_options.dart`** for real (this repo ships a
   placeholder that will compile but won't connect to anything):
   ```bash
   dart pub global activate flutterfire_cli
   flutterfire configure
   ```
5. **Seed daily questions.** Each document in the `daily_questions`
   collection is keyed by `yyyy-MM-dd` with a single `text` field —
   this is what guarantees every user in the world sees the same
   question. Seed a week or two ahead via the Firebase console or a
   small admin script.
6. **Platform setup**
   - iOS: add `GoogleService-Info.plist` via Xcode, enable Sign in with
     Apple capability, request camera/microphone usage strings in
     `Info.plist`.
   - Android: add `google-services.json`, request `CAMERA` and
     `RECORD_AUDIO` permissions in `AndroidManifest.xml`.
7. **Run**
   ```bash
   flutter run
   ```

## What's implemented (v1 scope, per spec)

- Onboarding (mission slides → "Begin Listening")
- Auth (Google / Apple / Email — email includes a full sign-in/sign-up form)
- Home screen: today's question, record CTA, listening progress,
  streak, listening score, yesterday's summary — nothing else
- Recorder: single take, 3-minute hard cap, no filters/retakes
- Listening Mode: one story at a time, fairness-first queue (stories
  under 20 unique listeners surface first), 6-minute daily goal, quiet
  completion state (no confetti)
- Listening Score + streak logic (both reflecting *and* listening are
  required to advance a streak)
- Profile: personal growth stats only — no followers/likes/analytics
- Native camera/mic/notification permission strings (iOS Info.plist,
  Android AndroidManifest.xml)
- A scheduled Cloud Function that writes a brand-new question to
  Firestore every night using Claude

## The nightly question generator

`functions/src/index.ts` contains `generateDailyQuestion`, a Cloud
Function scheduled for 11pm UTC daily. It:

1. Reads the last 30 days of questions from `daily_questions` so it
   never repeats itself.
2. Calls Claude with a system prompt encoding the spec's tone rules
   (vulnerability, reflection, curiosity, gratitude, hope, resilience,
   kindness, connection — never political, never assumes a specific
   life circumstance).
3. Writes the result to `daily_questions/{yyyy-MM-dd}` for the
   following day.
4. Falls back to a hand-picked question list if the API call fails,
   so the app is never left without a question.

There's also a callable function, `regenerateQuestionForDate`, for
manually backfilling or re-rolling a specific day from an admin tool.

**To deploy it:**
```bash
cd functions
npm install
firebase functions:secrets:set ANTHROPIC_API_KEY
npm run deploy
```

You'll need an Anthropic API key from console.anthropic.com. The
Firestore rules (`firestore.rules`) already lock `daily_questions`
writes to admin-only, so clients can never write their own question.

## Book cross-promotion

`lib/presentation/home/widgets/book_movement_link.dart` adds one
quiet, muted line at the very bottom of the home screen —
"Part of a larger movement · [Book Title]" — that expands into a
bottom sheet with a short blurb, a QR code, and a link out to Amazon.
Nothing shows unless someone taps it; it's intentionally the least
visible thing on the screen so it doesn't compete with the daily
ritual.

**Before shipping:** open `lib/core/constants/book_constants.dart` and
replace `amazonUrl` with your real Amazon listing link (and adjust the
title/tagline/blurb if you want different copy). This required adding
two new dependencies — `qr_flutter` and `url_launcher` — already in
`pubspec.yaml`.

## Native permissions

`ios/Runner/Info.plist` and `android/app/src/main/AndroidManifest.xml`
declare the camera/microphone strings the recorder needs and the
notification permission Android 13+ requires. If you've already run
`flutter create .` and have your own native scaffolding, merge the
`NSCameraUsageDescription` / `NSMicrophoneUsageDescription` keys and
the `<uses-permission>` lines into your existing files rather than
overwriting them outright.

## Deliberately not built (per spec)

Messaging, comments, likes, groups, followers, AI chat, or anything
that would turn this into another attention-driven feed.

## Notes on the fairness algorithm

`StoryRepositoryImpl.getNextStoriesToListenTo` queries stories with
`uniqueListenerCount < 20`, ordered ascending by that same count, so
the least-heard stories are always surfaced first. There is no
recency, popularity, or creator-quality signal in that query by
design — the spec's "Guaranteed Listening" system is about fairness,
not virality. You'll want a composite Firestore index on
`(uniqueListenerCount ASC, createdAt ASC)` once this is live.
