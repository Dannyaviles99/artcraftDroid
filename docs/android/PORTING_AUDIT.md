# Android ARM64 porting audit

Status: **foundation branch; not an Android build yet**.

Target: one ArtCraft codebase with a touch-first phone layout and a desktop-style tablet / external-display layout. Initial ABI: `arm64-v8a` (`aarch64-linux-android`). The first constrained device target is MediaTek Helio G81 with 4 GB RAM.

## Repository facts confirmed

- The native Rust UI is currently a binary at `artcraft-native/src/main.rs`.
- It starts through `eframe::run_native` and constructs `ArtcraftApp`.
- `artcraft-native/Cargo.toml` currently enables eframe's `wayland` and `x11` features.
- The native crate currently includes desktop integrations: `directories`, `open`, `rfd`, `rodio`, `arboard`, and `wry`.
- The workspace includes API client crates and a larger Tauri/TypeScript application. This port should reuse the Rust application and API clients where feasible, not attempt to package the desktop Linux window system on Android.

## Non-negotiable product goals

1. Preserve ArtCraft's real provider/model and generation workflows; do not replace the product with a demo screen.
2. Ship both UI modes in the same Android app:
   - Phone: touch-first navigation, compact panels, large hit targets, portrait and landscape.
   - Tablet / desktop mode: resizable or wide layout where Android permits it, side panels, keyboard and pointer support.
3. Keep business logic and task state shared between modes.
4. Treat 4 GB RAM as a constrained target: bounded image decoding, thumbnail and texture caches, limited concurrent previews, streaming file IO, and graceful memory-pressure handling.
5. Do not claim support for stylus pressure/tilt unless the device actually reports those values.

## Work completed on the foundation branch

- Added a compact bottom navigation for the four native workflows (Image, Video, Audio, Library) when the app window is narrower than 720 logical points.
- The desktop sidebar remains in use on wider windows; the same `ArtcraftApp` state and page implementations are used in both layouts.
- The content panel now preserves its outer inset when the phone layout hides the sidebar.

This is the first responsive-layout slice, not a complete mobile redesign. The create forms, dialogs, library picker, and media lightbox still need narrow-screen audits and touch-target adjustments.

## Main engineering blockers

### 1. Android application entry point

The current `main.rs` is a desktop `eframe::run_native` entry point. Android needs an Android-compatible eframe/winit application entry point and Android packaging. Confirm the exact eframe 0.36 Android integration API before selecting NativeActivity or GameActivity; do not introduce a speculative JNI bridge.

### 2. Split platform-specific dependencies

Move desktop-only integrations behind target-specific modules and Cargo target dependencies. The Android build must not attempt to compile Linux windowing backends or desktop browser/window integrations.

| Current integration | Android direction |
|---|---|
| `rfd` file dialogs | Storage Access Framework (SAF) |
| `arboard` clipboard | Android ClipboardManager |
| `open` external URLs | Android intents / Custom Tabs |
| `directories` | app-private Android directories |
| `wry` Midjourney login window | supported OAuth/deep-link flow or documented provider-specific alternative |
| `rodio` | verify Android audio backend; isolate behind an audio service |

### 3. SAF is URI-based

Android document selection returns content URIs, not ordinary filesystem paths. Refactor file operations to accept streams or a platform document abstraction. Do not convert content URIs into fake `PathBuf` values. Large assets must stream to bounded cache files rather than being loaded fully into memory.

### 4. Authentication

The current WRY login/cookie capture path cannot be assumed to work on Android. Do not extract cookies from another app or weaken authentication. Determine provider-supported OAuth, browser handoff, or deep-link options for each affected provider.

### 5. UI adaptation

Keep a single application state and shared workflows. Add a layout policy selected from available window size/input capabilities, with a user override. Audit fixed desktop window assumptions, hover-only controls, dense sidebars, context menus, keyboard shortcuts, and drag-and-drop.

## Recommended implementation sequence

1. **Build inventory:** identify every desktop-only API used by `artcraft-native` and its transitive internal crates. Initial source scan confirms WRY-based Midjourney login, `rfd` dialogs, `open` URL/folder launching, desktop home/cache paths, and `rodio` playback need platform handling.
2. **Platform boundary:** define narrow interfaces for document access, clipboard, external links, app paths, audio, and provider sign-in.
3. **Android entry point:** enable the documented Android eframe/winit backend for the locked dependency versions; add Android manifest and Gradle packaging.
4. **First launch:** build and install a minimal ARM64 app that starts the real `ArtcraftApp`, not a placeholder.
5. **Core workflow:** restore API configuration, provider authentication, model listing, generation submission, task polling, and result display.
6. **Files and library:** implement SAF URI streaming and persistent access grants.
7. **Responsive UI:** implement phone and tablet/desktop layouts over shared state.
8. **Device validation:** test on Helio G81/4 GB and at least one newer ARM64 Android device.

## Acceptance gates

- [ ] Android ARM64 debug build succeeds from a clean checkout.
- [ ] App launches on a physical ARM64 device.
- [ ] Existing ArtCraft account/provider authentication works through supported flows.
- [ ] Model listing and generation workflows use the real API clients.
- [ ] Generated images/videos can be viewed and saved through SAF.
- [ ] Phone and tablet/desktop layouts are both usable.
- [ ] Keyboard, mouse, touch, and available stylus input work.
- [ ] Repeated large previews do not cause unbounded memory growth.
- [ ] Desktop Linux build remains functional.

## Current limitation

This commit records the verified source audit and implementation plan. It does **not** yet add an Android launcher, compile the Rust app for Android, or produce an APK. Those require the platform boundary and a verified eframe Android entry point first.
