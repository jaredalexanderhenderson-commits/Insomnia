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

The app's features and default settings are unchanged. Claude and the
ChatGPT/Codex host remain in the upstream protected agent list. The duration
input calculation is split into intermediate integer values because Swift
6.1 could not type-check the original expression in reasonable time; its
validation and results are unchanged.

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

The same installation also retained an old
`usr/include/swift/module.modulemap` alongside `bridging.modulemap`. Both
define `SwiftBridging`, producing a duplicate-module error. A local Clang VFS
overlay hides the old file from the build. Explicit module compilation
passes this overlay to SDK dependency builds and avoids the repeated
implicit framework compilation that stalled the earlier attempts.

From the local checkout, the successful release build command is:

```bash
SWIFTPM_CUSTOM_LIBS_DIR="$PWD/../swiftpm-libs" \
  swift build -c release --product Insomnia --jobs 4 \
  --use-integrated-swift-driver --experimental-explicit-module-build \
  --scratch-path /tmp/insomnia-verify-build \
  -Xcc -ivfsoverlay -Xcc "$PWD/../swiftpm-libs/toolchain-overlay.json"
```

The local workaround files live beside the checkout, not in the app or the
fork. A healthy Swift installation does not need them. The signed app in
`../../outputs/Insomnia-macOS15.zip` uses the same bundle contents as
`scripts/build-app.sh`, including the sealed upstream recovery script.
The installer extracts it into a temporary folder before calling the
upstream installer. This avoids Finder metadata attached to bundles under
Documents by the file provider, which can make strict signature checks fail.
Strict signature verification also passed on a fresh extraction of the ZIP.

## Validation on macOS 15.5

Verified on the development Mac on October 12, 2026:

- Debug and optimized release builds completed with Swift 6.1 in Swift 6
  language mode. The arm64 executable declares macOS 15.0 as its minimum OS.
- The ad-hoc signed app passes strict code-signature verification and
  launches with an initialized, visible menu-bar item. It reads the lid,
  battery and thermal state and reconciles a clean temporary home.
- 36 targeted checks pass for duration input, presets, configuration
  persistence, unchanged defaults, session arithmetic and the actual
  display/keyboard guards and brightness read/write calls. Brightness
  writes used the existing levels.
- The compiled session manager passes start, extension, journal persistence
  and end/restoration checks with simulated power and recovery services. It
  also refuses a session when recovery cannot be armed and rolls back when
  disabling sleep fails. These checks do not change real power settings.
- Shell scripts pass macOS Bash syntax checks. The packaged recovery CLI
  rejects malformed input, the backstop succeeds with a clean temporary
  home, and lid-simulation code is absent from the release binary.

The full `swift test` suite could not run: these Command Line Tools do not
include XCTest, and full Xcode is not installed. Installation, privileged
power changes and a physical lid-close session still require verification
after the administrator completes the original installer. Automated and
startup checks do not establish closed-lid wakefulness.
