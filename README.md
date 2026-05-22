---
name: rokid-glass-dev
description: Build Android Go (YodaOS-Sprite) apps that run directly on Rokid Glasses hardware, including CXR‑S SDK integration and system input handling.
license: MIT
---

This skill teaches an AI coding agent how to **create, build, run, and debug Android applications that run directly on Rokid Glasses** (YodaOS‑Sprite, based on **Android Go**), including **UI constraints**, **device connection**, **system button/touch interactions**, and **CXR‑S SDK** integration for communication with the mobile Rokid app.

> Key assumption: “bare metal” development on Rokid Glasses is **standard Android app development** with additional platform constraints and SDK capabilities.

---

## What can this skill do

- Create a new **Android app (Kotlin)** project suitable for Rokid Glasses (Android Go).
- Apply Rokid Glasses **UI constraints** (design size **480 × 640**) and interaction guidelines.
- Configure Gradle to use Rokid’s Maven repository and integrate **CXR‑S SDK**:
  - `com.rokid.cxr:cxr-service-bridge:1.0-20250519.061355-45`
  - `minSdk >= 28`
- Implement **device connection status** monitoring via `CXRServiceBridge.StatusListener`.
- Implement **message subscribe / reply** patterns using `subscribe()` with `MsgCallback` and `MsgReplyCallback`.
- Implement **message sending** using `sendMessage()` with `Caps` and optional binary payload.
- Implement **system button / gesture** handling:
  - Ordered broadcast receiver for Rokid system actions (button, long press, two-finger gestures, AI start).
  - `onKeyDown/onKeyUp` for other key events such as `KEYCODE_BACK` and `KEYCODE_ENTER`.
- Provide **run/debug workflow**:
  - Connect using the dedicated dev cable.
  - Enable ADB via Rokid AI app.
  - Use screen mirroring (e.g., **scrcpy**) during development.
