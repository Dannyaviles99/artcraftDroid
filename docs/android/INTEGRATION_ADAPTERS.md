# Android integration adapters

This document is the implementation contract for replacing desktop-only integrations without
removing the existing desktop behavior. The Android work stays on
`android/arm64-foundation`; the desktop implementation remains the reference implementation.

## Non-negotiable design

- Keep provider clients, generation requests, model catalog, feed state, and page state in Rust.
- Put operating-system operations behind a small platform boundary.
- Android file selections are **document handles**, not filesystem paths. Never convert a
  `content://` URI into a fabricated `PathBuf`.
- Keep desktop integrations available on desktop targets.
- Do not report an Android integration as complete until it is wired into the app and built
  for `aarch64-linux-android`.

## Adapter contracts and migration order

### 1. App storage

Current entry point: `backend/session.rs::data_root` uses
`directories::UserDirs` and assumes a home directory.

Target contract:

- Resolve the app-private persistent directory from the Android activity's
  `filesDir` / native app internal data path.
- Resolve cache separately from Android `cacheDir`.
- Keep credentials, settings, and app-owned downloads under the private persistent root.
- Do not use shared external storage for session credentials.
- On desktop, preserve the current `~/Artcraft` behavior.

The Android activity handle must be passed into the Rust platform bootstrap. Do not use a
hard-coded package path or infer Android paths from `HOME`.

### 2. File selection and upload

Current entry point: `pages/common.rs::pick_files` returns `Vec<PathBuf>`; callers read
selected files with `std::fs::read` and pass paths to `Backend::upload_file`.
`overlays/characters.rs` follows the same path-based assumption.

Replace this with a platform-neutral selected-media value:

```rust
pub struct SelectedMedia {
    pub display_name: String,
    pub mime_type: Option<String>,
    pub source: MediaSource,
}

pub enum MediaSource {
    DesktopPath(std::path::PathBuf),
    AndroidDocumentUri(String),
}
```

The Android picker must use the Storage Access Framework
(`ACTION_OPEN_DOCUMENT`, with MIME filters for images, video/mp4, and audio). For multiple
selection use `EXTRA_ALLOW_MULTIPLE`. Persist read permission when the provider grants it.
Return display name and MIME type from `ContentResolver` metadata.

Uploads must consume a reader/stream (or a bounded temporary copy) rather than requiring
`PathBuf`. For Android document URIs, open the URI through `ContentResolver.openInputStream`.
Use a bounded app-cache copy only where an existing API truly requires a seekable file.
Delete temporary copies after upload. Preserve the current MP4 validation and image preview
behavior; derive file type from MIME/name metadata rather than a fake URI extension.

### 3. External URLs

Replace direct `open::that` call sites with a platform operation. Android uses
`ACTION_VIEW` (and Custom Tabs for browser-based sign-in); desktop keeps `open::that`.
Handle the no-browser/invalid-URL case and surface an app toast instead of silently failing.

### 4. Authentication / embedded web

The current Midjourney flow uses WRY to capture login cookies from an embedded webview.
Do not attempt to read another app's cookies or scrape a browser's private storage.

First inspect the provider's supported authentication flow. Prefer official OAuth/deep-link
callbacks or an approved provider web flow. If a provider requires an embedded user agent
and cookie handoff, implement only a provider-compliant flow and protect tokens with Android
Keystore. Keep unsupported sign-in methods explicitly unavailable with an explanation rather
than silently pretending sign-in succeeded.

### 5. Clipboard

Replace `arboard` calls behind the platform boundary. Android implementation uses
`ClipboardManager`; desktop retains `arboard`. Clipboard image support must be evaluated
separately from text support and must not be advertised until implemented.

### 6. Audio

Keep the app-level playback interface, but isolate `rodio` behind the platform boundary.
Verify the selected Android audio backend on a real ARM64 device. If `rodio` cannot provide
the required backend for the pinned version, use an Android-native Oboe/AAudio adapter.
Preserve play, pause, stop, completion, and error reporting.

## Android bridge lifecycle

The Android bridge is initialized from the eframe/winit Android entry point with the actual
activity handle. It owns pending Activity Result callbacks and dispatches their results back
to Rust. A picker request is asynchronous: Rust must not block the UI thread waiting for a
document selection.

The bridge must handle activity recreation, cancellation, revoked URI permissions, and
process death. Pending operations should fail cleanly if the activity is no longer available.

## Acceptance checks for each adapter

- Desktop behavior remains unchanged.
- Android behavior is exercised on `arm64-v8a`, not just host unit tests.
- Picker cancellation returns an empty/cancel result, not an error toast.
- Large media is streamed or copied with a strict size/memory policy.
- URI permission failures produce actionable errors.
- No credentials are stored in shared storage or logs.
- A feature is marked complete only after an end-to-end test through the real ArtCraft page.


## Implemented groundwork (2026-10-31)

The ArtCraft image and video file-upload endpoints now use asynchronous file streaming:
`tokio::fs::File` + `tokio_util::io::ReaderStream` + reqwest multipart
`Part::stream_with_length`. This removes the previous `std::fs::read` full-file allocation
from these two upload paths. It reduces peak memory pressure for large media on desktop and
provides a better base for Android's SAF-to-private-cache staging flow.

This is **not yet SAF support**: current UI callers still supply `PathBuf`, and Android
document URIs are not yet selected or opened by a native Activity Result bridge. The change
has been committed but has not yet been compiled or exercised on an Android device.
