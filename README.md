# Simple TV Launcher (SDK 22 Compatible)

A lightweight TV home launcher built for Android 5.1 Lollipop (API 22) and above, compatible with older Fire TV sticks and other legacy Android TV devices.

## Project Location

```
/home/pooja/Desktop/WORK/AI/fire-stick/cleanuptv/simple-launcher/
```

## Why This Was Created

The existing launcher at `cleanyourtv/cleanyourtv/launcher/` requires `minSdk 26` and uses APIs unavailable on Android 5.1 (API 22):

| API | Existing Launcher | Simple Launcher |
|-----|------------------|-----------------|
| `minSdk` | 26 | **22** |
| Date/time | `java.time.LocalDate` (API 26+) | `java.util.Calendar` (API 1+) |
| Broadcast receiver | `RECEIVER_NOT_EXPORTED` (API 26+) | Not used |
| TV Input framework | `TvInputManager` | Not used |
| Layout | RecyclerView + DynamicAnimation | GridLayout + ScrollView (framework) |
| Java version | 17 | 1.8 |
| Dependencies | AndroidX RecyclerView + DynamicAnimation | AndroidX RecyclerView only |

## Project Structure

```
simple-launcher/
├── build.gradle                      # Gradle project config
├── settings.gradle
├── gradle.properties
├── install_manual.sh                 # Build + install + launch script
├── simple-launcher.apk               # Pre-built signed APK (debug)
├── app/
│   ├── build.gradle                  # App module config (minSdk 22)
│   ├── proguard-rules.pro
│   └── src/main/
│       ├── AndroidManifest.xml       # HOME + LEANBACK_LAUNCHER intent filters
│       ├── java/com/simplelauncher/
│       │   ├── MainActivity.java     # Main activity: grid, clock, dock, D-pad nav
│       │   └── AppAdapter.java       # (legacy, no longer used)
│       ├── res/
│       │   ├── layout/
│       │   │   ├── activity_main.xml # Main layout: clock + grid + scroll
│       │   │   └── item_app.xml      # Individual app tile layout
│       │   ├── drawable/
│       │   │   ├── banner.xml
│       │   │   ├── tile_bg.xml
│       │   │   ├── tile_focused.xml
│       │   │   ├── tile_dock_bg.xml
│       │   │   └── tile_dock_focused.xml
│       │   ├── mipmap-*/ic_launcher.xml  # Launcher icon (mdpi, hdpi, xhdpi, xxhdpi)
│       │   └── values/
│       │       ├── colors.xml
│       │       ├── strings.xml
│       │       └── styles.xml
```

## Key Source Files

### `app/src/main/AndroidManifest.xml`
Declares the app as a home launcher with `android.intent.category.HOME` and `android.intent.category.LEANBACK_LAUNCHER` intent filters. Package: `com.simplelauncher`.

### `app/src/main/java/com/simplelauncher/MainActivity.java`
Main launcher activity. Features:
- Discovers installed apps via `PackageManager.queryIntentActivities()` (LEANBACK_LAUNCHER + LAUNCHER categories)
- Displays apps in a `GridLayout` with 6 columns (optimized for TV)
- D-pad navigation between tiles
- Click to launch an app
- Long-press to add/remove app from dock (max 5 dock slots)
- On-screen clock and date (updated every minute)
- Dock preference saved in `SharedPreferences`
- Back button is intercepted (does nothing on home)

### `app/src/main/res/layout/activity_main.xml`
Root layout: black background `FrameLayout` containing a `ScrollView` with:
- Clock (`TextClock`-style `TextView`)
- Date (`TextView`)
- `GridLayout` for app tiles (6 columns)
- Hint text at bottom

### `app/src/main/res/layout/item_app.xml`
Individual app tile: `FrameLayout` card with an `ImageView` icon (72dp) and `TextView` label (13sp, max 2 lines). Different drawable backgrounds for normal, focused, docked, and dock-focused states.

## Build Process

### Manual Build (Used)

Gradle 8.1.1 cannot run on Java 21 (class file version 65 unsupported), and Gradle 8.9 download was incomplete. The APK was built manually using Android SDK build tools:

```bash
# 1. Compile resources
aapt2 compile -o res/ --dir app/src/main/res

# 2. Link resources (generates R.java)
aapt2 link -o resources.apk --manifest app/src/main/AndroidManifest.xml \
    -I $ANDROID_HOME/platforms/android-34/android.jar \
    --auto-add-overlay --java gen/ --version-code 1 --version-name 1.0 \
    res/*.flat

# 3. Compile Java
javac -source 8 -target 8 -bootclasspath $ANDROID_HOME/platforms/android-34/android.jar \
    -cp gen/ -d obj/ app/src/main/java/com/simplelauncher/MainActivity.java

# 4. Convert to DEX
d8 --output dex/ --lib $ANDROID_HOME/platforms/android-34/android.jar obj/*.class

# 5. Package APK
cp resources.apk simple-launcher.apk
(cd dex && zip -r ../simple-launcher.apk classes.dex)

# 6. Align
zipalign -p 4 simple-launcher.apk simple-launcher-aligned.apk

# 7. Sign
apksigner sign --ks ~/.android/debug.keystore --ks-key-alias androiddebugkey \
    --ks-pass pass:android --key-pass pass:android \
    --out simple-launcher-signed.apk simple-launcher.apk
```

### Using `install_manual.sh`

The script at `simple-launcher/install_manual.sh` automates the entire build, install, and launch process:

```bash
cd /home/pooja/Desktop/WORK/AI/fire-stick/cleanuptv/simple-launcher
./install_manual.sh 192.168.0.118
```

What it does:
1. Verifies `ANDROID_HOME` and required build tools
2. Connects to the Fire Stick via ADB
3. Builds the APK from source (steps above)
4. Installs the APK (`adb install -r`)
5. Launches the app (`adb shell am start -n com.simplelauncher/.MainActivity`)

## Installation on Fire Stick

### Prerequisites
- Android SDK installed (`ANDROID_HOME` set, e.g., `~/Android/Sdk`)
- Platform tools (adb) available
- Build tools 34.0.0+ installed
- Platform 34 (`android-34`) installed

### Steps

1. **Enable ADB on Fire Stick**: Settings → My Fire TV → Developer Options → ADB Debugging ON
2. **Connect**: `adb connect 192.168.0.118:5555`
3. **Run the script**: `./install_manual.sh 192.168.0.118`

### Verification
```bash
# Check if package is installed
adb -s 192.168.0.118:5555 shell pm list packages | grep simplelauncher

# Check if app is running
adb -s 192.168.0.118:5555 shell dumpsys activity top | grep simplelauncher

# Launch manually
adb -s 192.168.0.118:5555 shell am start -n com.simplelauncher/.MainActivity
```

## Known Limitations

1. **Cannot set as default home launcher without root**: On Fire OS 5.1 (API 22), `cmd package set-home-activity` is not available, and `pm disable` on Amazon's launcher is blocked by SecurityException. The app can only be launched manually via ADB.

2. **No root access**: Without root, you cannot disable Amazon's stock launcher, so the Simple Launcher cannot become the default home screen automatically.

3. **Basic feature set**: This is a minimal launcher — no wallpaper support, no settings screen, no app library, no cinematic shelf, no animations beyond simple focus scaling.

## Files Modified During Development

| File | Change |
|------|--------|
| `app/src/main/AndroidManifest.xml` | Added `package="com.simplelauncher"` attribute (required by aapt2 link) |
| `app/src/main/java/com/simplelauncher/MainActivity.java` | Fixed missing `grid = findViewById(R.id.grid)` that caused NullPointerException on resume |
| `app/src/main/res/layout/item_app.xml` | Reduced tile size: 160×200dp → 110×140dp, icon 96dp → 72dp, label 15sp → 13sp |
| `app/src/main/res/layout/activity_main.xml` | Reduced padding 24dp → 16dp, clock 56sp → 40sp, date 18sp → 16sp, columns 5 → 6 |
| `install_manual.sh` | Expanded from launch-only to full build + install + launch script |

## Future Enhancements

- Add settings activity for configurable columns, dock size, colors
- Add wallpaper support
- Add app library / hide apps feature
- Add focus animations and transitions
- Add screensaver / clock mode
- Add leanback banner support for better TV integration

