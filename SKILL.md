---
name: rokid-glass-dev
description: Build Android Go (YodaOS-Sprite) apps that run directly on Rokid Glasses hardware, including CXR‑S SDK integration and system input handling.
license: MIT
---

# rokid-glass-dev — Skill Guide

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

---

## When to use this skill

Use this skill when you need to:

- Build an app that **runs directly on Rokid Glasses** (not just a phone companion app).
- Add Rokid Glasses specific capabilities:
  - CXR‑S ↔ mobile communication
  - System button / touchpad / gesture event handling
  - Glasses UI sizing and interaction constraints
- Produce ready-to-run code: Gradle config, manifest, Kotlin classes, and simple UI (Views or Jetpack Compose).

---

## Platform constraints (must-follow)

### OS / device constraints

- Rokid Glasses run **YodaOS‑Sprite**, based on **Android Go**.
- Follow **Android Go** constraints (lighter memory/CPU footprint; avoid heavy background work; keep APK lean).
- Ensure:
  - `minSdk >= 28` (required by the CXR‑S SDK referenced here).

### UI / screen

- Design reference size: **480 × 640 px**.
- Prefer:
  - ConstraintLayout / Compose with responsive layout
  - scalable text and spacing (don’t hardcode large phone-like paddings)
  - high-contrast UI and large touch targets (AR/Glasses context)
- When implementing UI, use dp/sp (not raw px) and test on device via mirroring.

### System interactions (do not break)

YodaOS‑Sprite defines some interactions with the mobile Rokid Glasses app **and they cannot be changed** in bare metal development:

- Long-press right-temple touchpad → enter Rokid AI app module
- Double-tap right-temple button → **Back**
- Top-right-temple button tap → take photo
- Top-right-temple button long-press → record video
- Certain wake words trigger features (details may change in future docs)

Your app should **coexist** with these behaviors. If you intercept events, do so carefully and avoid trapping the user with no way to navigate.

---

## Development environment & device setup

### Prerequisites

- A computer capable of Android development with USB support
- Android Studio (or another Android IDE)
- Rokid Glasses device
- **Dedicated development cable** (the default package may include only a charging cable; dev cable is required for data/ADB)

### Enable ADB on the glasses

ADB must be enabled **via the Rokid AI mobile app**.

### Connect the device

- The charging contacts on the **left temple** also serve as data contacts.
- Connect using the supplied charging & data cable (development cable required).

### Debugging & mirroring

- You can use screen mirroring tools such as **scrcpy** to view the glasses display during development.

---

## Project setup (recommended defaults)

### Recommended stack

- Kotlin + Android Gradle Plugin (AGP)
- Jetpack Compose (optional) OR XML views
- Keep dependencies minimal (Android Go friendliness)

### Manifest & features

- Avoid declaring unnecessary hardware features that would exclude installation.
- Request only required permissions.

---

## CXR‑S SDK integration (1.0)

### 1) Configure Rokid Maven repository

In `settings.gradle.kts`, add the Rokid Maven repository:

```kotlin
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        maven { url = uri("https://maven.rokid.com/repository/maven-public/") }
        mavenCentral()
    }
}
```

### 2) Add dependency + minSdk

In `build.gradle.kts`:

```kotlin
android {
    defaultConfig {
        minSdk = 28
    }
}

dependencies {
    implementation("com.rokid.cxr:cxr-service-bridge:1.0-20250519.061355-45")
}
```

### 3) Connection status listener

```kotlin
private val TAG = "StatusListener"
private val cxrBridge = CXRServiceBridge()

private val statusListener = object : CXRServiceBridge.StatusListener {
    override fun onConnected(name: String, type: Int) {
        Log.d(TAG, "Connected to $name type=$type")
    }

    override fun onDisconnected() {
        Log.d(TAG, "Disconnected")
    }

    override fun onARTCStatus(health: Float, reset: Boolean) {
        Log.d(TAG, "ARTC health=${(health * 100).toInt()}% reset=$reset")
    }
}

fun setStatusListener() {
    cxrBridge.setStatusListener(statusListener)
}
```

### 4) Subscribe to messages (no-reply)

```kotlin
private val cxrBridge = CXRServiceBridge()

private val msgCallback = object : CXRServiceBridge.MsgCallback {
    override fun onReceive(name: String, args: Caps, value: ByteArray?) {
        Log.i("MessageSubscribe", "Received name=$name argsSize=${args.size()} valueBytes=${value?.size}")
    }
}

fun subscribeExample() {
    cxrBridge.subscribe("glass_test", msgCallback)
}
```

### 5) Subscribe to messages (with reply)

```kotlin
private val cxrBridge = CXRServiceBridge()

private val replyCallback = object : CXRServiceBridge.MsgReplyCallback {
    override fun onReceive(name: String, args: Caps, value: ByteArray?, reply: Reply?) {
        Log.d("MessageSubscribe", "Received name=$name argsSize=${args.size()} valueBytes=${value?.size}")
        val ret = Caps().apply { write("Received Message and Reply") }
        reply?.end(ret)
    }
}

fun subscribeReplyExample() {
    cxrBridge.subscribe("glass_test", replyCallback)
}
```

### 6) Send messages (Caps only / Caps + binary)

Caps only:

```kotlin
fun sendCapsMessageExample(cxrServiceBridge: CXRServiceBridge) {
    val args = Caps().apply {
        write("send_message")
        writeUInt32(5)
    }

    val result = cxrServiceBridge.sendMessage("message_channel", args)
    Log.d("send_message", "result=$result")
}
```

Caps + binary:

```kotlin
fun sendBinaryMessageExample(cxrServiceBridge: CXRServiceBridge, data: ByteArray) {
    val args = Caps().apply {
        write("send_message")
        writeUInt32(5)
    }

    val result = cxrServiceBridge.sendMessage(
        "message_channel",
        args,
        data,
        /*offset*/ 0,
        /*size*/ data.size
    )
    Log.d("send_message", "result=$result")
}
```

### 7) Caps data structure tips

- `Caps` is a typed container used to serialize structured data.
- Prefer simple, versioned protocols between mobile and glasses:
  - first field: message version / type
  - subsequent fields: parameters

Example parse helper:

```kotlin
private fun parseCapsValue(value: Caps.Value): String = when (value.type()) {
    Caps.Value.TYPE_STRING -> "String: ${value.getString()}"
    Caps.Value.TYPE_INT32 -> "Int32: ${value.getInt()}"
    Caps.Value.TYPE_UINT32 -> "UInt32: ${value.getInt()}"
    Caps.Value.TYPE_INT64 -> "Int64: ${value.getLong()}"
    Caps.Value.TYPE_UINT64 -> "UInt64: ${value.getLong()}"
    Caps.Value.TYPE_FLOAT -> "Float: ${value.getFloat()}"
    Caps.Value.TYPE_DOUBLE -> "Double: ${value.getDouble()}"
    Caps.Value.TYPE_BINARY -> "Binary: ${value.getBinary().size} bytes"
    Caps.Value.TYPE_OBJECT -> "Caps Object"
    else -> "Unsupported type"
}
```

---

## System input handling on Rokid Glasses

Rokid provides **system button events** as **ordered broadcasts**. You can intercept them by registering a high-priority receiver and calling `abortBroadcast()`.

### Ordered broadcast receiver (reference)

```kotlin
interface KeyReceiverListener {
    fun onReceive(keyType: KeyType)
}

enum class KeyType(val action: String) {
    CLICK("com.android.action.ACTION_SPRITE_BUTTON_CLICK"),
    BUTTON_DOWN("com.android.action.ACTION_SPRITE_BUTTON_DOWN"),
    BUTTON_UP("com.android.action.ACTION_SPRITE_BUTTON_UP"),
    DOUBLE_CLICK("com.android.action.ACTION_SPRITE_BUTTON_DOUBLE_CLICK"),
    AI_START("com.android.action.ACTION_AI_START"),
    LONG_PRESS("com.android.action.ACTION_SPRITE_BUTTON_LONG_PRESS"),
    ACTION_TWO_FINGER_SINGLE_TAP("com.android.action.ACTION_TWO_FINGER_SINGLE_TAP"),
    ACTION_TWO_FINGER_DOUBLE_TAP("com.android.action.ACTION_TWO_FINGER_DOUBLE_TAP"),
    ACTION_TWO_FINGER_SWIPE_FORWARD("com.android.action.ACTION_TWO_FINGER_SWIPE_FORWARD"),
    ACTION_TWO_FINGER_SWIPE_BACK("com.android.action.ACTION_TWO_FINGER_SWIPE_BACK"),
    ACTION_SETTINGS_KEY("com.android.action.ACTION_SETTINGS_KEY")
}

class KeyReceiver : BroadcastReceiver() {
    var listener: KeyReceiverListener? = null

    override fun onReceive(context: Context?, intent: Intent?) {
        val action = intent?.action ?: return
        val type = KeyType.entries.firstOrNull { it.action == action } ?: return
        listener?.onReceive(type)

        // Note: Some system actions (e.g., “back/exit”) may still be reserved by system UX.
        abortBroadcast()
    }
}
```

Register receiver (Activity):

```kotlin
private val keyReceiver = KeyReceiver().apply {
    listener = object : KeyReceiverListener {
        override fun onReceive(keyType: KeyType) {
            Log.d("KeysActivity", "system event: $keyType")
        }
    }
}

@SuppressLint("UnspecifiedRegisterReceiverFlag")
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)

    // Keep screen on
    window.addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)

    registerReceiver(
        keyReceiver,
        IntentFilter().apply {
            KeyType.entries.forEach { addAction(it.action) }
            priority = 100
        }
    )
}
```

### Other keys via KeyEvent

```kotlin
@SuppressLint("GestureBackNavigation")
override fun onKeyDown(keyCode: Int, event: KeyEvent?): Boolean {
    Log.d("KeysActivity", "onKeyDown: $keyCode")
    return when (keyCode) {
        KeyEvent.KEYCODE_BACK -> true // intercept back if appropriate
        KeyEvent.KEYCODE_ENTER -> true
        else -> super.onKeyDown(keyCode, event)
    }
}
```

**Best practice:** avoid blocking the user from leaving the app. If you intercept back events, provide an in-app exit route or confirm dialog.

---

## Build / run / debug checklist (agent-ready)

When generating an app for Rokid Glasses, always ensure:

1. **Gradle repo** includes `https://maven.rokid.com/repository/maven-public/`
2. **minSdk = 28** (or higher) if using the provided CXR‑S SDK
3. UI designed for **480 × 640**
4. Handle reserved system gestures/buttons gracefully
5. Logging uses `Log.d/i/e` and can be inspected via `adb logcat`
6. Device debug steps are documented:
   - connect dev cable
   - enable ADB via Rokid AI app
   - deploy from Android Studio (Run/Debug)
   - optional mirroring with scrcpy

---

## Common pitfalls (avoid)

- Hardcoding phone-like UI (too large margins / tiny text / wrong aspect assumptions).
- Adding heavyweight dependencies (not friendly to Android Go).
- Assuming all system interactions can be overridden (some are reserved).
- Forgetting `minSdk >= 28` when adding the SDK dependency.
- Not documenting the “enable ADB via Rokid AI app” step (device won’t appear to Android Studio otherwise).

---

## Supporting references (uploaded images)

- UI design guidelines screenshot: `/.uploads/835ba029-a347-4631-bfb3-fbe6795bd730_rokid-design-guidelines.png`
- Maven settings screenshot: `/.uploads/51f61f26-b8e3-4814-a743-e07e5f4676b8_mavenSettings.png`

