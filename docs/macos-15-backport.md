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

On macOS 15, `plutil -type` prints a missing-key diagnostic to stdout before
exiting with status 1. The backstop and uninstaller's `type_of` helpers now
emit output only after a successful lookup, preserving their contract of
returning an empty string for missing keys. Without this change, the recovery
script's array scans loop indefinitely while holding the recovery lock,
blocking lid actions and timer cleanup. The lock implementation is unchanged.

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
- Missing-key regression checks pass for both recovery scripts with the
  actual macOS 15 `plutil`. A valid active-session backstop check completes
  in under 0.1 seconds and releases its lock. Forced recovery clears the
  session and journal using simulated power commands.

The full `swift test` suite could not run: these Command Line Tools do not
include XCTest, and full Xcode is not installed. The original installer has
completed, the app runs from `~/Applications`, and the loaded recovery agent
matches the installed app's pinned signature and completes its scheduled
runs successfully. A live two-minute session confirmed `SleepDisabled 1`.
The first lid-close test exposed the `plutil` issue above; after releasing
the stuck recovery process, the app retried its pending timer end and
restored `SleepDisabled 0`. The corrected recovery script is installed and
the repeated two-minute live test passed on battery power:

- Sleep was disabled for the active session. The power log records no
  sleep/wake transition during the test.
- The lid was closed for 21 seconds. Display and keyboard light were turned
  off, speakers muted, six Slack processes paused, and Low Power Mode
  enabled. Codex and Claude were not frozen.
- Reopening the lid resumed all six processes, restored audio and both
  brightness levels, and switched Low Power Mode off. No recovery-lock
  refusal occurred during the patched test.
- The timer ended at the saved deadline, restored `SleepDisabled 0`,
  removed the session file and left a clean recovery journal. The recovery
  agent continues to complete its scheduled checks with exit status 0.

macOS denied notification authorization during these sessions. Session
status and warnings remain available in the menu and log; native notification
banners were not validated. Battery-floor, critical thermal and extended
unattended sessions were not hardware-tested.
