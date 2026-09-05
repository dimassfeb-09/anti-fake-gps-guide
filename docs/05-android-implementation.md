# 5. Android Implementation

Copy-ready snippets. Request only the permissions you need, and explain to the
user why.

## Report model

```kotlin
data class LocationReport(
    val latitude: Double,
    val longitude: Double,
    val accuracy: Float,
    val altitude: Double,
    val speed: Float,          // m/s
    val bearing: Float,        // degrees
    val provider: String,      // "fused" / "gps" / "network" / "passive"
    val clientTimeMs: Long,
    val isMockup: Boolean,     // Location.isFromMockProvider
    val gpsEnabled: Boolean,   // LocationManager.isProviderEnabled(GPS)
    val lastGpsFixTime: String,
    val prevUpdateStatus: PrevUpdateStatus,
)

data class PrevUpdateStatus(
    val isSuccess: Boolean,
    val errorMessage: String,
)
```

## Build a single report

```kotlin
fun buildReport(loc: Location, mgr: LocationManager): LocationReport {
    val gpsEnabled = mgr.isProviderEnabled(LocationManager.GPS_PROVIDER)
    return LocationReport(
        latitude = loc.latitude,
        longitude = loc.longitude,
        accuracy = loc.accuracy,
        altitude = loc.altitude,
        speed = loc.speed,
        bearing = loc.bearing,
        provider = loc.provider ?: "unknown",
        clientTimeMs = System.currentTimeMillis(),
        isMockup = if (Build.VERSION.SDK_INT >= 18) loc.isFromMockProvider else false,
        gpsEnabled = gpsEnabled,
        lastGpsFixTime = lastGpsFixString(mgr), // format as "yyyy-MM-dd HH:mm:ss"
        prevUpdateStatus = lastUpdateStatus,    // track your own upload success/failure
    )
}
```

## Independent re-check (separate module)

```kotlin
fun anyProviderMocked(mgr: LocationManager): Boolean =
    mgr.getProviders(true).any { name ->
        mgr.getLastKnownLocation(name)?.isFromMockProvider == true
    }
```

## Play Integrity (call on login / new session)

Verify the response on your server, not on-device:

```kotlin
// Pseudocode — actual API is com.google.android.play:integrity
val token: String = requestIntegrityToken() // on-device call
// → POST token to your server → server calls Play API to verify
// → server checks: app recognized, not sideloaded, device not tampered
```

Tips:

- Batch environment fingerprinting (airplane/VPN/USB/CPU/package-added) once
  per session, not per fix — it saves battery and avoids permission fatigue.
- Use `FusedLocationProviderClient` for accuracy, fall back to
  `LocationManager` for the provider-level mock re-check.
