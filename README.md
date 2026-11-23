# soil-eight

Android library for reading data from the 8-in-1 soil sensor MIIKA (temperature, moisture, conductivity, pH, nitrogen, phosphorus, potassium, fertility) USB port (OTG)

> ⚠️ **License / Usage Restriction**
> This package is provided for **non-commercial use only**.
> - ❌ Commercial use is **not allowed** 
> - ❌ Modifying, decompiling, or redistributing the source code is **not allowed**.
> - ✅ You may use the compiled package as-is in recruitment purpose or personal non-commercial projects.

> Contact me at mhamzahr@gmail.com if you need a commercial license

---

## Installation

### 1. Add the GitHub Packages repository

This package is hosted on GitHub Packages, which requires authentication even for public packages.

In your **project-level** `settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven("https://jitpack.io")
        maven {
            url = uri("https://maven.pkg.github.com/hamric-id/soil-eight-Android")
            credentials {
                username = providers.gradleProperty("gpr.user").orNull
                password = providers.gradleProperty("gpr.key").orNull
            }
        }
    }
}
```

### 2. Add your GitHub credentials

In `~/.gradle/gradle.properties` (your **user home** folder, not the project):

```properties
gpr.user=your_github_username
gpr.key=ghp_your_personal_access_token
```
Create your token at GitHub → Settings → Developer settings → Personal access tokens (Classic), then checklist `read:packages` scope

### 3. Add the dependency

In your **module-level** `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.hamric:soil-eight:1.0.0")
}
```

Check the [Packages page](https://github.com/hamric-id/soil-eight-Android/packages) for the latest version number.

---

## Usage

### Step 1 — Create the sensor instance (via the factory)

```kotlin
val sensor: SoilSensorApi = SoilSensorFactory.create()
```

Always create it through `SoilSensorFactory.create()`, not the implementation class directly.

### Step 2 — Start the USB service

Call this once, typically in `onCreate()` / when your screen starts:

```kotlin
sensor.startUSBservice(context)
```

This registers the USB broadcast receiver and checks for an already-attached device.

### Step 3 — Observe the sensor status

```kotlin
lifecycleScope.launch {
    sensor.status.collect { status ->
        when (status) {
            SensorStatus.NO_SENSOR -> {
                // Sensor cable is unplugged / no USB device detected
            }
            SensorStatus.DISCONNECTED -> {
                // Sensor is plugged in, but not yet connected.
                // You must call connect() to start reading.
                sensor.connect(context) { success ->
                    if (success) {
                        // ready to read
                    }
                }
            }
            SensorStatus.CONNECTING -> {
                // Permission/connection in progress
            }
            SensorStatus.CONNECTED -> {
                // Ready — call readAll() or read() here
            }
            SensorStatus.DISCONNECTING -> {
                // disconnect() was called, closing in progress
            }
        }
    }
}
```

**SensorStatus meaning summary:**

| SensorStatus | Meaning |
|---|---|
| `NO_SENSOR` | Sensor is unplugged (no USB device found) |
| `DISCONNECTED` | Sensor is plugged in, but not connected yet — call `connect()` |
| `CONNECTING` | Requesting USB permission / opening the port |
| `CONNECTED` | Ready to read data |
| `DISCONNECTING` | `disconnect()` was called, closing in progress |

### Step 4 — Read data (only when status is `CONNECTED`)

Read all 8 parameters at once:

```kotlin
sensor.readAll { data ->
    if (data != null) {
        Log.d("Soil", "Temp: ${data.temperature}, Moisture: ${data.moisture}")
    }
}
```

Or read a single parameter:

```kotlin
sensor.read(SensorParameter.PH) { value ->
    Log.d("Soil", "pH: $value")
}
```

> ⏱ **Minimum read interval: 1000ms.**
> Do not call `readAll()` / `read()` faster than once per second — the sensor communicates over a slow serial link and faster polling can cause read failures or timeouts.

```kotlin
lifecycleScope.launch {
    while (isActive) {
        if (sensor.status.value == SensorStatus.CONNECTED) {
            sensor.readAll { data -> /* update UI */ }
        }
        delay(1000) // minimum interval
    }
}
```

### Step 5 — Disconnect and stop the service

Call this when your screen/app is destroyed, to release the USB connection and unregister the receiver:

```kotlin
override fun onDestroy() {
    sensor.disconnect()
    sensor.stopUSBservice()
    super.onDestroy()
}
```

---

## Full example

```kotlin
class MainActivity : AppCompatActivity() {

    private val sensor: SoilSensorApi = SoilSensorFactory.create()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        sensor.startUSBservice(this)

        lifecycleScope.launch {
            sensor.status.collect { status ->
                when (status) {
                    SensorStatus.DISCONNECTED -> sensor.connect(this@MainActivity) {}
                    SensorStatus.CONNECTED -> startReading()
                    else -> {}
                }
            }
        }
    }

    private fun startReading() {
        lifecycleScope.launch {
            while (isActive && sensor.status.value == SensorStatus.CONNECTED) {
                sensor.readAll { data ->
                    // handle data
                }
                delay(1000)
            }
        }
    }

    override fun onDestroy() {
        sensor.disconnect()
        sensor.stopUSBservice()
        super.onDestroy()
    }
}
```

---

## API Reference

| Method | Description |
|---|---|
| `SoilSensorFactory.create()` | Creates a new sensor instance |
| `startUSBservice(context)` | Registers USB listeners and checks for a connected device |
| `stopUSBservice()` | Unregisters USB listeners — call on destroy |
| `connect(context, onResult)` | Requests permission (if needed) and opens the serial port |
| `readAll(onResult)` | Reads all 8 parameters at once (`EightSoilData`) |
| `read(param, onResult)` | Reads a single `SensorParameter` |
| `disconnect()` | Closes the serial port |
| `status: StateFlow<SensorStatus>` | Observable connection status |

---

## Troubleshooting

### Sensor never leaves `NO_SENSOR`

- **Check your phone supports USB OTG.** Not all Android phones do — if OTG isn't supported in hardware, the phone can never detect the USB sensor/adapter, regardless of this library.
- **Check OTG is enabled in settings.** On some brands (Oppo, Vivo, Realme, and similar), OTG support has a toggle that **automatically turns itself off after ~20 minutes** of inactivity. If it was working before but stopped, this is almost always why. Re-enable it manually, usually found under:
  - **Settings → Additional Settings / Security / Developer Options → OTG** (path/wording varies by brand and OS version)

### Permission was granted, but status stays `DISCONNECTED` / never reaches `CONNECTED`

- On some device brands, after granting the USB permission dialog on first plug-in, the port still fails to open correctly 
- **Workaround:** unplug and re-plug the USB sensor once after granting permission if it not connected more than 3 sec. This is a known quirk on certain brands/ROMs, not a bug in your app — once re-plugged, it connects normally and subsequent sessions don't need this extra step unless the device detaches again.
