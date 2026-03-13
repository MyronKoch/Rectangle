# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is Rectangle?

Rectangle is a macOS window management app (Swift, AppKit) based on Spectacle. It provides keyboard shortcuts and drag-to-snap functionality for resizing/positioning windows. Bundle ID: `com.knollsoft.Rectangle`. Minimum macOS 10.15+.

## Build & Run

```bash
# Build via Xcode CLI (CI uses macos-26)
xcodebuild -project Rectangle.xcodeproj -scheme Rectangle archive CODE_SIGN_IDENTITY="-" -archivePath build/Rectangle.xcarchive

# Open in Xcode (preferred for development)
open Rectangle.xcodeproj
```

Dependencies (Sparkle, MASShortcut) are managed via Swift Package Manager - Xcode resolves them automatically. MASShortcut uses rxhanson's fork at github.com/rxhanson/MASShortcut.

**macOS < 26 build note:** The Liquid Glass icon causes build failure on older macOS. Delete "Asset Catalog Other Flags" in build settings to fix locally, but don't commit that change.

## Testing

Tests are in `RectangleTests/` but are scaffold-only (no meaningful test cases). Testing is manual - the app requires Accessibility permissions and real window manipulation that can't be easily unit tested.

## Architecture Overview

### Execution Flow (keyboard shortcut to window move)

```
MASShortcut captures key event
  -> Posts NSNotification with ExecutionParameters
  -> ShortcutManager receives notification
  -> MultiWindowManager/TodoManager check for special handling
  -> WindowManager.execute(parameters)
    -> AccessibilityElement.getFrontWindowElement() (AXUIElement wrapper)
    -> ScreenDetection.detectScreens() for current/adjacent displays
    -> WindowCalculationFactory maps WindowAction -> Calculation instance
    -> Calculation.calculate(params) -> WindowCalculationResult (new rect)
    -> GapCalculation applies configured gaps
    -> WindowMover chain: StandardWindowMover -> BestEffortWindowMover (fallback)
    -> Records action to WindowHistory for repeat/restore tracking
```

### Snap Flow (drag to screen edge)

```
SnappingManager monitors global mouse events
  -> Detects cursor proximity to screen edges/corners
  -> FootprintWindow shows visual overlay of target rect
  -> On mouse release, posts same action as keyboard shortcut path
```

### Key Design Patterns

- **Notification-driven:** All actions flow through `NSNotification`. `WindowAction` enum posts notifications; `ShortcutManager` subscribes.
- **Strategy pattern (WindowMover):** Chain of `StandardWindowMover` -> `BestEffortWindowMover` (fallback for quirky apps). Also `CenteringFixedSizedWindowMover` and `QuantizedWindowMover` (character-grid apps like iTerm2).
- **Factory pattern:** `WindowCalculationFactory` is a static registry mapping every `WindowAction` case to a `Calculation` instance.
- **Calculation protocol:** `Calculation` protocol -> `WindowCalculation` base class -> 76 specific implementations in `WindowCalculation/`. Each computes a target `CGRect` from current window rect, screen frame, and action type.

### Core Files

| File | Role |
|------|------|
| `AppDelegate.swift` | Main orchestrator - initializes managers, builds menus, handles lifecycle |
| `WindowAction.swift` | Enum with 100+ cases (leftHalf, maximize, etc.), default shortcuts, action metadata |
| `WindowManager.swift` | Executes window actions - coordinates detection, calculation, and movement |
| `ShortcutManager.swift` | Binds MASShortcut keyboard shortcuts, subscribes to action notifications |
| `AccessibilityElement.swift` | Wraps AXUIElement - gets/sets window frame, position, size, role |
| `ScreenDetection.swift` | Multi-display detection and screen ordering |
| `Defaults.swift` | Preferences via NSUserDefaults with typed wrappers (BoolDefault, FloatDefault, JSONDefault, etc.) |
| `WindowCalculation.swift` | Base `Calculation` protocol, `WindowCalculation` class, `WindowCalculationFactory` |
| `SnappingManager.swift` | Drag-to-edge snap detection and footprint overlay |
| `ApplicationToggle.swift` | Ignore/whitelist app logic (unregisters shortcuts when ignored app is frontmost) |

### Key Directories

| Directory | Contents |
|-----------|----------|
| `Rectangle/WindowCalculation/` | 76 calculation files - one per snap position (halves, thirds, fourths, sixths, eighths, ninths) |
| `Rectangle/Snapping/` | Drag-to-snap detection, FootprintWindow overlay, CompoundSnapArea for multi-pane snapping |
| `Rectangle/WindowMover/` | Movement strategy implementations |
| `Rectangle/Utilities/` | CGRect/NSScreen extensions, EventMonitor, Debounce, etc. |
| `Rectangle/PrefsWindow/` | Settings UI controllers |
| `Rectangle/TodoMode/` | Todo Mode sidebar management |
| `Rectangle/MultiWindow/` | Tile-all and cascade-all operations |
| `RectangleLauncher/` | Login-item helper app (macOS < 13 only; newer macOS uses native LaunchAgent) |

## Adding a New Window Action

1. Add a case to the `WindowAction` enum in `WindowAction.swift` (assign an explicit raw value)
2. Create a calculation class in `WindowCalculation/` implementing `calculateRect(_:)` -> `RectResult`
3. Register the calculation in `WindowCalculationFactory.calculationsByAction`
4. Add default keyboard shortcut in `WindowAction`'s `defaultShortcuts` property
5. The action name string (used in URL scheme) goes in the `name` computed property

## Preferences System

- All preferences stored in `NSUserDefaults` (`com.knollsoft.Rectangle.plist`)
- Typed wrappers in `Defaults.swift`: `BoolDefault`, `FloatDefault`, `StringDefault`, `IntDefault`, `JSONDefault<T>`, `OptionalBoolDefault`
- Hidden preferences configurable via `defaults write com.knollsoft.Rectangle ...` (documented in `TerminalCommands.md`)
- Import/export as JSON: `~/Library/Application Support/Rectangle/RectangleConfig.json`
- Values load at app startup - restart required after terminal changes

## URL Scheme

`rectangle://execute-action?name=[action-name]` triggers any window action programmatically. Example:
```bash
open -g "rectangle://execute-action?name=left-half"
```

## Coordinate System Note

macOS uses a flipped coordinate system (origin at bottom-left) while accessibility APIs use top-left origin. The codebase uses `.screenFlipped` conversions throughout to reconcile this. Be careful with CGRect math - always check which coordinate space you're in.

## Coding Conventions

- Match existing style (per CONTRIBUTING.md)
- Calculations return `RectResult` wrapping a `CGRect` plus optional `SubWindowAction`
- Repeat-command cycling (e.g., first third -> center third -> last third) is handled in individual calculations via `isRepeatedCommand()` checks
- Window history tracking (`AppDelegate.windowHistory`) enables restore and repeat-cycle behavior
