# RustDesk Guide

## Project Layout

### Directory Structure
* `src/` Rust app
* `src/server/` audio / clipboard / input / video / network
* `src/platform/` platform-specific code
* `src/ui/` legacy Sciter UI (deprecated)
* `flutter/` current UI
* `libs/hbb_common/` config / proto / shared utils
* `libs/scrap/` screen capture
* `libs/enigo/` input control
* `libs/clipboard/` clipboard
* `libs/hbb_common/src/config.rs` all options

### Key Components
- **Remote Desktop Protocol**: Custom protocol implemented in `src/rendezvous_mediator.rs` for communicating with rustdesk-server
- **Screen Capture**: Platform-specific screen capture in `libs/scrap/`
- **Input Handling**: Cross-platform input simulation in `libs/enigo/`
- **Audio/Video Services**: Real-time audio/video streaming in `src/server/`
- **File Transfer**: Secure file transfer implementation in `libs/hbb_common/`

### UI Architecture
- **Legacy UI**: Sciter-based (deprecated) - files in `src/ui/`
- **Modern UI**: Flutter-based - files in `flutter/`
  - Desktop: `flutter/lib/desktop/`
  - Mobile: `flutter/lib/mobile/`
  - Shared: `flutter/lib/common/` and `flutter/lib/models/`

## Rust Rules

* Avoid `unwrap()` / `expect()` in production code.
* Exceptions:

  * tests;
  * lock acquisition where failure means poisoning, not normal control flow.
* Otherwise prefer `Result` + `?` or explicit handling.
* Do not ignore errors silently.
* Avoid unnecessary `.clone()`.
* Prefer borrowing when practical.
* Do not add dependencies unless needed.
* Keep code simple and idiomatic.

## Tokio Rules

* Assume a Tokio runtime already exists.
* Never create nested runtimes.
* Never call `Runtime::block_on()` inside Tokio / async code.
* Do not hide runtime creation inside helpers or libraries.
* Do not hold locks across `.await`.
* Prefer `.await`, `tokio::spawn`, channels.
* Use `spawn_blocking` or dedicated threads for blocking work.
* Do not use `std::thread::sleep()` in async code.

## Editing Hygiene

* Change only what is required.
* Prefer the smallest valid diff.
* Do not refactor unrelated code.
* Do not make formatting-only changes.
* Keep naming/style consistent with nearby code.

## Viper Custom Build Notes

This branch carries Viper-specific customizations that must be preserved during
future upstream cherry-picks or rebases.

### Branding

The public app name was changed from `RustDesk` to `SuporteViper` while keeping
Viper-specific app icons. Do not rename executables,
package ids, URI schemes, icon resource names, or install paths unless explicitly
requested; keeping them as `rustdesk` avoids breaking build scripts, deep links,
desktop files, and packaging assumptions.

Important branding files:

* `libs/hbb_common/src/config.rs` sets `APP_NAME` to `SuporteViper`.
* `Cargo.toml`, `libs/portable/Cargo.toml`, and `flutter/windows/runner/Runner.rc`
  carry Windows/package metadata.
* `flutter/android/app/src/main/AndroidManifest.xml`,
  `flutter/android/app/src/main/res/values/strings.xml`, and Android Kotlin
  services carry Android labels, notification names, and visible service text.
* `flutter/macos/Runner/Configs/AppInfo.xcconfig` and
  `flutter/ios/Runner/Info.plist` carry Apple display names.
* `res/rustdesk.desktop`, `res/rustdesk-link.desktop`,
  `appimage/AppImageBuilder-*.yml`, `flatpak/com.rustdesk.RustDesk.metainfo.xml`,
  and `res/msi/Package/Language/Package.en-us.wxl` carry Linux/AppImage/Flatpak/MSI
  display metadata.
* App icons were replaced with the Viper red/white logo across Android mipmaps,
  iOS `AppIcon.appiconset`, Windows `.ico`, macOS `.icns`, Linux PNG/SVG assets,
  and `flutter/assets/icon.*`. Preserve these generated icon assets during
  upstream cherry-picks.

When resolving conflicts, preserve `SuporteViper` in visible labels,
notifications, product names, service display names, and installer text. It is
acceptable for technical ids such as `rustdesk`, `com.rustdesk.RustDesk`,
`com.carriez.flutter_hbb`, and `rustdesk://` to remain unchanged.

### Rendezvous Build Secrets

The custom rendezvous server and public key are injected at compile time from
GitHub Actions secrets:

* `RENDEZVOUS_SERVER`
* `RS_PUB_KEY`

The implementation is in `libs/hbb_common/src/config.rs` using `option_env!`,
with the upstream RustDesk server/key as local-build fallbacks. The submodule
build script `libs/hbb_common/build.rs` declares
`cargo:rerun-if-env-changed=RENDEZVOUS_SERVER` and
`cargo:rerun-if-env-changed=RS_PUB_KEY` so Cargo rebuilds when secrets change.

The reusable workflow `.github/workflows/flutter-build.yml` exports both
secrets. Tag builds use `.github/workflows/flutter-tag.yml`, which calls that
workflow with `secrets: inherit`, so preserving `flutter-build.yml` is enough for
future tag builds.

### Submodule And Cherry-pick Guidance

`libs/hbb_common` is a Git submodule. Branding and rendezvous changes exist both
inside that submodule and in the root repository. During cherry-picks, verify
both worktrees:

* Root repo: `git status --short`
* Submodule: `git -C libs/hbb_common status --short`
* Root diff: `git diff -- .github/workflows/flutter-build.yml Cargo.toml flutter res appimage flatpak src libs/portable`
* Submodule diff: `git -C libs/hbb_common diff -- build.rs src/config.rs src/platform/mod.rs`

If upstream changes touch RustDesk branding or rendezvous defaults, re-apply the
Viper choices after conflict resolution and then commit the submodule change
before committing the updated submodule pointer in the root repo.

### Legacy RustDesk Service Cleanup

When installing the Viper build, the installer must stop, disable, and remove
the previous `rustdesk.service` / `RustDesk` service before installing the new
service. Preserve this behavior in future packaging changes.

Important files:

* Debian packages: `res/DEBIAN/preinst` and `res/DEBIAN/postinst`.
* RPM/SUSE packages: `res/rpm.spec`, `res/rpm-flutter.spec`,
  `res/rpm-suse.spec`, and `res/rpm-flutter-suse.spec`.
* Pacman packages: `res/pacman_install`.
* Windows MSI: `res/msi/Package/Components/RustDesk.wxs` schedules
  `TryStopDeleteOldRustDeskService` before service creation, and
  `res/msi/Package/Fragments/CustomActions.wxs` maps that action to the existing
  `TryStopDeleteService` custom action.

This cleanup is intentionally tolerant (`|| true` / ignored MSI return) so a
fresh install without any old RustDesk service still succeeds.

### Software Update

Keep software update disabled for the Viper build until a Viper-owned update
endpoint/release flow exists. The upstream RustDesk update API returns RustDesk
release URLs and must not be used by `SuporteViper`.

Preserve these guards:

* `src/common.rs`: `do_check_software_update()` returns immediately for custom
  clients and clears `SOFTWARE_UPDATE_URL`.
* `src/updater.rs`: automatic/manual updater checks return immediately for
  custom clients.
