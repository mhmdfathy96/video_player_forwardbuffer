# Forked `video_player` federation — forward-buffer cap

These are local forks of the official `video_player` packages, patched to add a
**Dart-configurable forward-buffer cap**. Upstream exposes no knob for this and
defaults to aggressive read-ahead (ExoPlayer ~50s; AVPlayer buffers most of the
file on iOS), which wastes bandwidth when a user abandons or seeks away. See the
long-standing, still-unimplemented requests:
flutter/flutter#40931, #141511, #163918.

Wired into the app via `dependency_overrides` (path) in the root `pubspec.yaml`.

## Base versions (upstream, from pub.dev)

| Package                          | Base version |
|----------------------------------|--------------|
| video_player                     | 2.11.1       |
| video_player_platform_interface  | 6.6.0        |
| video_player_android             | 2.9.1        |
| video_player_avfoundation        | 2.9.0        |

## Public API added

`VideoPlayerOptions(forwardBufferDuration: Duration?)` — null = platform default
(unchanged upstream behavior); a value caps network read-ahead.

```dart
VideoPlayerController.networkUrl(
  uri,
  videoPlayerOptions: VideoPlayerOptions(
    forwardBufferDuration: const Duration(seconds: 10),
  ),
);
```

## Changes per package (the only diffs vs upstream)

- **video_player_platform_interface** (`lib/video_player_platform_interface.dart`)
  - `VideoPlayerOptions`: add `Duration? forwardBufferDuration`.
  - `VideoCreationOptions`: add `Duration? forwardBufferDuration`.
- **video_player** (`lib/video_player.dart`)
  - `initialize()`: thread `videoPlayerOptions?.forwardBufferDuration` into the
    `VideoCreationOptions` it builds.
- **video_player_android**
  - `pigeons/messages.dart` + generated `lib/src/messages.g.dart` &
    `android/.../Messages.kt`: add `forwardBufferDurationMs` (int?/Long?) to
    `CreationOptions`.
  - `lib/src/android_video_player.dart`: pass
    `forwardBufferDurationMs: options.forwardBufferDuration?.inMilliseconds`.
  - `VideoPlayer.java`: add `cappedLoadControl(Long)` helper (DefaultLoadControl
    with clamped playback thresholds; null → ExoPlayer default).
  - `TextureVideoPlayer.java` / `PlatformViewVideoPlayer.java`: accept the value,
    `setLoadControl(...)` only when non-null.
  - `VideoPlayerPlugin.java`: pass `options.getForwardBufferDurationMs()` into both.
- **video_player_avfoundation**
  - `pigeons/messages.dart` + generated `lib/src/messages.g.dart`,
    `.../messages.g.h`, `.../messages.g.m`: add `forwardBufferDurationMs` to
    `(FVP)CreationOptions`.
  - `lib/src/avfoundation_video_player.dart`: pass the value (ms).
  - `FVPVideoPlayerPlugin.m` `playerItemWithCreationOptions:`: set
    `item.preferredForwardBufferDuration = ms/1000` when provided.

> The generated pigeon files were hand-edited (single nullable field appended at
> the end of each `CreationOptions` codec list — existing indices unchanged).
> The `pigeons/messages.dart` sources are updated too, so a real `dart run pigeon`
> regeneration for the upstream PR will reproduce these diffs cleanly.

## Upstream plan

Open a PR to `flutter/packages` adding `forwardBufferDuration`. If accepted,
delete these forks and the `dependency_overrides`, returning to pub versions.
