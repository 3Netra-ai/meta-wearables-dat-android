# Meta Wearables DAT SDK — Android Studio AI Instructions

> Full API reference: https://wearables.developer.meta.com/llms.txt?full=true
> Developer docs: https://wearables.developer.meta.com/docs/develop/

This file gives Gemini in Android Studio full context for the Meta Wearables Device Access
Toolkit (DAT SDK). Use this knowledge when answering questions, generating code, or suggesting
fixes for any DAT SDK integration in this project.

---

## Architecture

The SDK is organized into three Gradle modules:

| Module | Purpose |
|--------|---------|
| `mwdat-core` | Device discovery, registration, permissions, device selectors |
| `mwdat-camera` | `StreamSession`, `VideoFrame`, photo capture |
| `mwdat-mockdevice` | `MockDeviceKit` — testing without hardware |

---

## Supported Devices

| Device | Generation | SDK Support | Key Capability |
|--------|-----------|-------------|----------------|
| Ray-Ban Meta | **Gen 1** | v0.1.0+ | Camera, audio, open-ear speakers |
| Meta Ray-Ban Display | **Gen 2** | v0.4.0+ | Camera, audio, built-in display |

Both generations share the same streaming API. Gen 2 adds a display; display-output APIs are
planned for a future SDK version.

---

## Gradle Setup

### `settings.gradle.kts`

```kotlin
val localProperties =
    Properties().apply {
        val localPropertiesPath = rootDir.toPath() / "local.properties"
        if (localPropertiesPath.exists()) {
            load(localPropertiesPath.inputStream())
        }
    }

dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven {
            url = uri("https://maven.pkg.github.com/facebook/meta-wearables-dat-android")
            credentials {
                username = ""
                password = System.getenv("GITHUB_TOKEN") ?: localProperties.getProperty("github_token")
            }
        }
    }
}
```

### `libs.versions.toml`

```toml
[versions]
mwdat = "0.5.0"

[libraries]
mwdat-core       = { group = "com.meta.wearable", name = "mwdat-core",       version.ref = "mwdat" }
mwdat-camera     = { group = "com.meta.wearable", name = "mwdat-camera",     version.ref = "mwdat" }
mwdat-mockdevice = { group = "com.meta.wearable", name = "mwdat-mockdevice", version.ref = "mwdat" }
```

### `app/build.gradle.kts`

```kotlin
dependencies {
    implementation(libs.mwdat.core)
    implementation(libs.mwdat.camera)
    implementation(libs.mwdat.mockdevice)
}
```

---

## AndroidManifest.xml

```xml
<manifest>
    <uses-permission android:name="android.permission.BLUETOOTH" />
    <uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
    <uses-permission android:name="android.permission.INTERNET" />

    <application>
        <!-- Use 0 in Developer Mode; production apps get an ID from Wearables Developer Center -->
        <meta-data
            android:name="com.meta.wearable.mwdat.APPLICATION_ID"
            android:value="0" />

        <!-- Optional: disable analytics -->
        <meta-data
            android:name="com.meta.wearable.mwdat.ANALYTICS_OPT_OUT"
            android:value="true" />

        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.VIEW" />
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />
                <data android:scheme="myexampleapp" />   <!-- replace with your URL scheme -->
            </intent-filter>
        </activity>
    </application>
</manifest>
```

---

## SDK Initialization

Call `Wearables.initialize(context)` once in `Application.onCreate()` before any other SDK call.
Calling SDK APIs before initialization yields `WearablesError.NOT_INITIALIZED`.

```kotlin
import com.meta.wearable.dat.core.Wearables

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        Wearables.initialize(this)
    }
}
```

---

## Registration & Permissions

### Register with Meta AI

```kotlin
// Opens the Meta AI app for user approval
Wearables.startRegistration(activity)

// Observe state changes
lifecycleScope.launch {
    Wearables.registrationState.collect { state ->
        when (state) {
            is RegistrationState.Registered   -> { /* can request permissions */ }
            is RegistrationState.Unregistered -> { /* prompt user to register */ }
            else                              -> { /* Registering / Unregistering */ }
        }
    }
}

// Unregister
Wearables.startUnregistration(activity)
```

### Camera permission

```kotlin
// Check current status
val result = Wearables.checkPermissionStatus(Permission.CAMERA)
result.onSuccess { status ->
    if (status == PermissionStatus.Granted) { /* start streaming */ }
}

// Request permission with Activity Result API
private val permissionsResultLauncher =
    registerForActivityResult(Wearables.RequestPermissionContract()) { status ->
        if (status == PermissionStatus.Granted) startStream()
    }

fun requestCameraPermission() {
    permissionsResultLauncher.launch(Permission.CAMERA)
}
```

---

## Device Discovery

```kotlin
// Observe all known devices (Gen 1 + Gen 2)
lifecycleScope.launch {
    Wearables.devices.collect { deviceIdentifiers ->
        // Set<DeviceIdentifier>
    }
}

// Read device metadata
lifecycleScope.launch {
    Wearables.devicesMetadata[deviceId]?.collect { metadata ->
        val name        = metadata.name
        val firmware    = metadata.firmwareVersion
        val compat      = metadata.compatibility  // COMPATIBLE | DEVICE_UPDATE_REQUIRED
    }
}
```

### Device compatibility check (Gen 1 & Gen 2)

```kotlin
lifecycleScope.launch {
    Wearables.devicesMetadata[deviceId]?.collect { metadata ->
        if (metadata.compatibility == DeviceCompatibility.DEVICE_UPDATE_REQUIRED) {
            showError("${metadata.name} needs a firmware update")
        }
    }
}
```

---

## Device Selection

### Auto-select (recommended)

```kotlin
// Filters out incompatible devices by default
val selector = AutoDeviceSelector()

// Include all devices regardless of compatibility
val selector = AutoDeviceSelector(filter = { true })

// Prefer Gen 2 over Gen 1
val selector = AutoDeviceSelector(
    rank = { deviceId ->
        val metadata = Wearables.devicesMetadata[deviceId]?.value
        when (metadata?.deviceType) {
            DeviceType.META_RAYBAN_DISPLAY -> 0   // Gen 2 preferred
            DeviceType.RAYBAN_META         -> 1   // Gen 1 fallback
            else                           -> 2
        }
    }
)
```

### Select a specific device

```kotlin
val selector = SpecificDeviceSelector(deviceIdentifier = knownDeviceId)
```

---

## Camera Streaming

### Start a stream session

```kotlin
val session = Wearables.startStreamSession(
    context = context,
    deviceSelector = AutoDeviceSelector(),
    streamConfiguration = StreamConfiguration(
        videoQuality = VideoQuality.MEDIUM,  // 504x896
        frameRate = 24,
    ),
)
```

### Resolution options

| Quality | Size |
|---------|------|
| `VideoQuality.HIGH`   | 720 × 1280 |
| `VideoQuality.MEDIUM` | 504 × 896  |
| `VideoQuality.LOW`    | 360 × 640  |

Valid frame rates: `2`, `7`, `15`, `24`, `30` FPS.
Lower resolution + lower frame rate = higher per-frame quality (less Bluetooth compression).

### Observe stream state

```kotlin
// State machine: STARTING → STARTED → STREAMING → STOPPING → STOPPED → CLOSED
lifecycleScope.launch {
    session.state.collect { state ->
        when (state) {
            StreamSessionState.STREAMING -> { /* frames flowing */ }
            StreamSessionState.STOPPED  -> { /* release resources */ }
            StreamSessionState.CLOSED   -> { /* session done */ }
            else -> { /* transitioning */ }
        }
    }
}
```

### Receive video frames

```kotlin
lifecycleScope.launch {
    session.videoStream.collect { frame ->
        // VideoFrame contains bitmap data — display in ImageView / Canvas / Compose
        imageView.setImageBitmap(frame.bitmap)
    }
}
```

### Photo capture

```kotlin
session.capturePhoto()
    .onSuccess { photoData ->
        val bytes = photoData.data   // JPEG bytes
    }
    .onFailure { error ->
        // CaptureError: DeviceDisconnected | NotStreaming | CaptureInProgress | CaptureFailed
    }
```

---

## Kotlin Patterns

- Use `suspend` functions for async operations — no callbacks
- Use `StateFlow` / `Flow` for state observation
- Use `DatResult<T, E>` for error handling — not exceptions

### DatResult

```kotlin
val result = Wearables.someOperation()
result.fold(
    onSuccess = { value -> /* handle value */ },
    onFailure = { error -> /* handle error */ }
)

// Partial handling
result.onSuccess { value -> /* ... */ }
result.onFailure { error -> /* ... */ }
```

Do **not** use `getOrThrow()` — always handle both paths.

---

## Session Lifecycle

```kotlin
lifecycleScope.launch {
    Wearables.getDeviceSessionState(deviceId).collect { state ->
        when (state) {
            SessionState.RUNNING -> { /* perform live work */ }
            SessionState.PAUSED  -> { /* hold — do not restart */ }
            SessionState.STOPPED -> { /* free resources */ }
        }
    }
}
```

When `PAUSED`: streams stop but the connection stays alive. Wait for `RUNNING` or `STOPPED` —
never restart during `PAUSED`.

Closing hinges = Bluetooth disconnects = session forced to `STOPPED`.
Opening hinges restores Bluetooth but does **not** auto-restart the session.

---

## MockDeviceKit (Testing)

Use MockDeviceKit to test without physical glasses.

### Setup

```kotlin
val mockDeviceKit = MockDeviceKit.getInstance(context)

// Attach fake stack (auto-initializes Wearables). Defaults to Registered state.
mockDeviceKit.enable()

// Or start unregistered to test registration flows:
// mockDeviceKit.enable(MockDeviceKitConfig(initiallyRegistered = false))
```

### Simulate Gen 1 glasses lifecycle

```kotlin
val device = mockDeviceKit.pairRaybanMeta()   // Ray-Ban Meta (Gen 1)
device.powerOn()
device.unfold()
device.don()    // wearing

// ... test streaming ...

device.doff()
device.fold()
device.powerOff()
```

### Mock camera feed

```kotlin
val camera = device.getCameraKit()
camera.setCameraFeed(videoUri)        // h.265 video for streaming
camera.setCapturedImage(imageUri)     // JPEG/PNG for capturePhoto()
```

### Teardown

```kotlin
mockDeviceKit.disable()   // Unpairs all devices, restores real SDK stack
```

### Instrumentation test base class

```kotlin
open class MockDeviceKitTestCase<T : Any>(
    private val activityClass: Class<T>
) {
    @get:Rule val scenarioRule = ActivityScenarioRule(activityClass)

    protected lateinit var mockDeviceKit: MockDeviceKitInterface
    protected lateinit var targetContext: Context

    @Before open fun setUp() {
        targetContext = InstrumentationRegistry.getInstrumentation().targetContext
        mockDeviceKit = MockDeviceKit.getInstance(targetContext)
        InstrumentationRegistry.getInstrumentation().uiAutomation.run {
            executeShellCommand("pm grant ${targetContext.packageName} android.permission.BLUETOOTH_CONNECT")
            executeShellCommand("pm grant ${targetContext.packageName} android.permission.CAMERA")
        }
        mockDeviceKit.enable()
    }

    @After open fun tearDown() {
        mockDeviceKit.disable()
    }
}
```

---

## Naming Conventions

| Suffix | Purpose | Example |
|--------|---------|---------|
| `*Manager` | Long-lived resource management | `RegistrationManager` |
| `*Session` | Short-lived flow component | `StreamSession` |
| `*Result` | `DatResult` type aliases | `RegistrationResult` |
| `*Error` | Error sealed interfaces | `WearablesError`, `CaptureError` |

Methods: `get*`, `set*`, `check*`, `request*`, `observe*`

---

## Key Imports

```kotlin
import com.meta.wearable.dat.core.Wearables                     // SDK entry point
import com.meta.wearable.dat.core.selectors.AutoDeviceSelector
import com.meta.wearable.dat.core.selectors.SpecificDeviceSelector
import com.meta.wearable.dat.core.types.*                        // DeviceIdentifier, Permission, etc.
import com.meta.wearable.dat.camera.StreamSession
import com.meta.wearable.dat.camera.types.*                      // VideoFrame, StreamConfiguration, etc.
import com.meta.wearable.dat.mockdevice.MockDeviceKit
import com.meta.wearable.dat.mockdevice.api.MockDeviceKitInterface
```

---

## Version Compatibility

| SDK | Meta AI App | Ray-Ban Meta (Gen 1) | Meta Ray-Ban Display (Gen 2) |
|-----|-------------|----------------------|------------------------------|
| 0.5.0 | See [docs](https://wearables.developer.meta.com/docs/version-dependencies) | See docs | See docs |
| 0.4.0 | V254 | V20 | V21 |
| 0.3.0 | V249 | V20 | — |

---

## Debugging Checklist

- [ ] `Wearables.initialize(context)` called in `Application.onCreate()`
- [ ] Developer Mode enabled in Meta AI app (Settings → Your glasses → Developer Mode)
- [ ] Meta AI app updated to a compatible version
- [ ] Glasses firmware updated
- [ ] Internet available (registration requires connectivity)
- [ ] `BLUETOOTH_CONNECT` permission granted at runtime
- [ ] `APPLICATION_ID` set in `AndroidManifest.xml`
- [ ] Correct URL scheme in `<intent-filter>`

### Common symptom → cause table

| Symptom | Likely cause |
|---------|-------------|
| Registration completes but device never connects | Developer Mode not enabled |
| `StreamSession` stuck in `STARTED`, never `STREAMING` | Camera permission not granted |
| `WearablesError.NOT_INITIALIZED` | `Wearables.initialize()` not called |
| No frames received | Device out of range or disconnected |
| Gen 2 device not selected | SDK < 0.4.0; upgrade to 0.4.0+ |

---

## Links

- [Android API Reference](https://wearables.developer.meta.com/docs/reference/android/dat/0.5)
- [Developer Documentation](https://wearables.developer.meta.com/docs/develop/)
- [Version dependencies](https://wearables.developer.meta.com/docs/version-dependencies)
- [Known issues](https://wearables.developer.meta.com/docs/knownissues)
- [GitHub Repository](https://github.com/facebook/meta-wearables-dat-android)
- [CameraAccess sample](https://github.com/facebook/meta-wearables-dat-android/tree/main/samples)
