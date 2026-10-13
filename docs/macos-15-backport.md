# macOS 15 backport

This fork targets Apple Silicon Macs running macOS 15 or later and Swift 6.1.
It lowers the package deployment target and bundle minimum OS. Swift 6 remains
the language mode selected by the Swift 6.1 package manifest.

DisplayServices is allowed on macOS 15 after a local check on macOS 15.5
(24F74), arm64, on October 12, 2026:

- Built-in display 1 reported brightness control available.
- GetBrightness returned success and brightness 0.5.
- SetBrightness to the existing value returned success.
- SetBrightness to 0 returned success; GetBrightness confirmed 0.
- Restoration to 0.5 returned success; GetBrightness confirmed 0.5.
- All six KeyboardBrightnessClient runtime method encodings matched the
  upstream macOS 26 measurements. Upstream runtime checks remain enabled.

Codex Pulse (`com.jaredhenderson.codexpulse`) is added to the protected agent
list. Claude and the ChatGPT/Codex host were already protected.

This does not establish that closed-lid sessions, recovery, keyboard brightness
writes, battery thresholds, or thermal behavior have passed hardware testing
on macOS 15. Test a short supervised session before leaving a closed Mac
running, and keep it on a ventilated surface.

## Building

Use `scripts/build-app.sh --output /path/to/output` with Swift 6.1 or newer.
The existing installer installs the signed app, recovery LaunchAgent and
restricted pmset sudoers rule; an administrator password is required.

On the development Mac, Command Line Tools had stale 2024 private SwiftPM
interfaces alongside 2025 public interfaces. A temporary local copy of
ManifestAPI with those stale private interfaces removed, selected through
SWIFTPM_CUSTOM_LIBS_DIR, allowed package loading. No system toolchain files
were changed. This workaround is separate from the OS backport.
