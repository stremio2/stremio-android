# Building the Android APK on Ubuntu (x86_64 debug, from source)

These steps build the app **and** its native Rust libraries (`libstream_server.so`,
`libstremio_core_kotlin.so`) from source, then install the APK on an emulator or device.
They are the exact steps that produced a working x86_64 debug APK on Ubuntu (24.04-style
setup, Java 21, NDK 29). Expect about 30 minutes on a fast machine the first time, mostly
vcpkg building libtorrent. Later Kotlin-only changes rebuild in about 20 seconds.

Disk: plan for roughly 25 GB (SDK, NDK, vcpkg packages, Rust target directories).

All paths below assume everything lives under `~/stremio`. Adjust `ROOT` if you prefer.

## 1. System packages

```bash
sudo apt-get update
sudo apt-get install -y \
  git curl zip unzip tar build-essential \
  pkg-config ninja-build bison flex nasm \
  autoconf automake libtool perl python3 cmake \
  protobuf-compiler openjdk-21-jdk
java -version   # must report 21
```

`pkg-config` is required by vcpkg; without it the first package (`libiconv`) fails with
"Could not find pkg-config". `ninja-build`, `bison`, `flex`, `nasm` are used by vcpkg
ports. `protobuf-compiler` provides `protoc` for the core bridge.

## 2. Android SDK and NDK

Install the Android command-line tools (download "Command line tools only" from
https://developer.android.com/studio and unpack so that
`~/Android/Sdk/cmdline-tools/latest/bin/sdkmanager` exists), then:

```bash
export ANDROID_HOME=$HOME/Android/Sdk
export ANDROID_SDK_ROOT=$ANDROID_HOME
SDKM=$ANDROID_HOME/cmdline-tools/latest/bin/sdkmanager

yes | $SDKM --licenses
$SDKM "platform-tools" "emulator" "platforms;android-37.0" "build-tools;36.0.0" \
      "ndk;29.0.13846066" "cmake;3.22.1"
```

The app uses `compileSdk 37`, `minSdk 24`, and NDK `29.0.13846066` (pinned in
`app/build.gradle.kts`). Gradle can download missing SDK platforms itself once licenses
are accepted.

Add to `~/.bashrc` (open a new shell afterwards):

```bash
export ANDROID_HOME=$HOME/Android/Sdk
export ANDROID_SDK_ROOT=$ANDROID_HOME
export ANDROID_NDK_HOME=$ANDROID_HOME/ndk/29.0.13846066
export PATH=$PATH:$ANDROID_HOME/platform-tools:$ANDROID_HOME/emulator
```

## 3. Rust toolchain

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --profile minimal
source "$HOME/.cargo/env"
rustup target add x86_64-linux-android aarch64-linux-android
cargo install cargo-ndk
```

## 4. Get the source (with submodules)

```bash
mkdir -p ~/stremio && cd ~/stremio
git clone --recurse-submodules git@github.com:stremio2/stremio-android.git
cd stremio-android
```

Submodules: `stream-server` (Rust streaming server) and `streamio-core-kotlin`
(Stremio core bridge).

## 5. Build `libstream_server.so` (slow part: libtorrent via vcpkg)

The server is built with `--features libtorrent`, which needs libtorrent and OpenSSL
built for Android by vcpkg at the commit pinned in the release workflow.

```bash
export ROOT=$HOME/stremio
export SS=$ROOT/stremio-android/stream-server
export VCPKG_DISABLE_METRICS=1

# 5a. vcpkg at the pinned commit
git clone https://github.com/microsoft/vcpkg.git $ROOT/vcpkg
git -C $ROOT/vcpkg checkout 84bab45d415d22042bd0b9081aea57f362da3f35
$ROOT/vcpkg/bootstrap-vcpkg.sh -disableMetrics

# 5b. Android triplet (x86_64; the repo only ships Linux/Windows triplets)
cat > $SS/triplets/x64-android.cmake <<'EOF'
set(VCPKG_TARGET_ARCHITECTURE x64)
set(VCPKG_CRT_LINKAGE dynamic)
set(VCPKG_LIBRARY_LINKAGE static)
set(VCPKG_CMAKE_SYSTEM_NAME Android)
set(VCPKG_CMAKE_SYSTEM_VERSION 24)
set(VCPKG_MAKE_BUILD_TRIPLET "--host=x86_64-linux-android")
set(VCPKG_CMAKE_CONFIGURE_OPTIONS -DANDROID_ABI=x86_64)
EOF

# 5c. Build libtorrent + OpenSSL (about 70 packages, 15-20 minutes)
cd $SS
$ROOT/vcpkg/vcpkg install --triplet x64-android \
  --x-install-root=$ROOT/vcpkg_installed-x86_64 \
  --overlay-triplets=triplets --overlay-ports=vcpkg-overlays

# 5d. Build the server library
source $HOME/.cargo/env
TR=$ROOT/vcpkg_installed-x86_64/x64-android
export LIBCLANG_PATH=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/lib
cd $SS/server
VCPKG_ROOT=$ROOT/vcpkg VCPKG_INSTALLED_DIR=$ROOT/vcpkg_installed-x86_64 \
VCPKGRS_TRIPLET=x64-android PKG_CONFIG_ALLOW_CROSS=1 \
PKG_CONFIG_PATH=$TR/lib/pkgconfig OPENSSL_DIR=$TR \
cargo ndk --target x86_64-linux-android --platform 24 build --release \
  --features libtorrent --no-default-features

mkdir -p ~/stremio/stremio-android/app/src/main/jniLibs/x86_64
cp $SS/target/x86_64-linux-android/release/libstream_server.so \
   ~/stremio/stremio-android/app/src/main/jniLibs/x86_64/
```

Two gotchas that cost time:

- **Do not set `PKG_CONFIG_SYSROOT_DIR`** (the release workflow does). With a local
  `pkg-config` it doubles the include path and `libtorrent/session.hpp` is "not found".
- **Set `LIBCLANG_PATH`** to the NDK's `lib` directory. Without it, `async-sevenz` and
  `async-rar` fail in bindgen with "Unable to find libclang".

## 6. Build `libstremio_core_kotlin.so`

```bash
source $HOME/.cargo/env
cd ~/stremio/stremio-android/streamio-core-kotlin/stremio-core-kotlin
cargo ndk --target x86_64-linux-android --platform 21 build --release
mkdir -p src/androidMain/jniLibs/x86_64
cp ../target/x86_64-linux-android/release/libstremio_core_kotlin.so \
   src/androidMain/jniLibs/x86_64/
```

(About 3 minutes. The Cargo build fetches `Stremio/stremio-core` from GitHub, so this step
needs network access.)

## 7. Build the debug APK

```bash
cd ~/stremio/stremio-android
chmod +x gradlew
./gradlew :app:assembleDebug -Pandroid.injected.build.abi=x86_64
```

Output: `app/build/intermediates/apk/debug/app-x86_64-debug.apk` (about 126 MB). The
`injected.build.abi` property limits the build to x86_64 and makes Gradle mark the APK
test-only, so install it with `-t` (next step).

If Gradle warns that `sdk.dir ... Directory does not exist`, that comes from a committed
`local.properties` in the core submodule; the build still uses `ANDROID_HOME`.

## 8. Emulator and install

```bash
# one-time: create a phone AVD with an x86_64 image
$ANDROID_HOME/cmdline-tools/latest/bin/sdkmanager "system-images;android-35;google_apis;x86_64"
echo no | $ANDROID_HOME/cmdline-tools/latest/bin/avdmanager create avd \
  -n pixel_api35 -k "system-images;android-35;google_apis;x86_64" -d pixel

# KVM makes the emulator fast
ls -l /dev/kvm   # your user must be in the 'kvm' group

emulator -avd pixel_api35 -no-boot-anim &
adb wait-for-device
adb install -r -t app/build/intermediates/apk/debug/app-x86_64-debug.apk
adb shell am start -n com.stremio.mobile/.MainActivity
```

Healthy start-up log line: `adb logcat | grep JniStreamingServer` shows
`Streaming server ready at http://127.0.0.1:11470`.

## 9. Real devices (arm64) and Android TV

For an arm64 phone or TV box, repeat steps 5 and 6 for `aarch64-linux-android` and copy the
libraries to `jniLibs/arm64-v8a/`. Use this triplet (`arm64-android.cmake`) and the vcpkg
install root `vcpkg_installed-arm64`:

```cmake
set(VCPKG_TARGET_ARCHITECTURE arm64)
set(VCPKG_CRT_LINKAGE dynamic)
set(VCPKG_LIBRARY_LINKAGE static)
set(VCPKG_CMAKE_SYSTEM_NAME Android)
set(VCPKG_CMAKE_SYSTEM_VERSION 24)
set(VCPKG_MAKE_BUILD_TRIPLET "--host=aarch64-linux-android")
set(VCPKG_CMAKE_CONFIGURE_OPTIONS -DANDROID_ABI=arm64-v8a)
```

Then build with `-Pandroid.injected.build.abi=arm64-v8a`, and sideload with
`adb connect <tv-ip>:5555 && adb install -r -t <apk>`.

## 10. Rebuilding after code changes

Kotlin-only changes (player, UI) do not touch the Rust libraries. Re-run only step 7 and
the `adb install` line.

## 11. Run the unit tests

```bash
./gradlew :app:testDebugUnitTest --tests "com.stremio.mobile.player.*" \
  -Pandroid.injected.build.abi=x86_64
```

## Not covered here

- Signed release APKs (see `.github/workflows/release-apk.yml` for the signing inputs).
- All four ABIs in one APK. The release workflow builds each ABI in a matrix; locally you
  would repeat steps 5 and 6 per ABI with matching triplets.
- Making the build independent of GitHub. Step 6 still downloads `Stremio/stremio-core`
  and `Stremio/local-search`; mirror those repos and vendor the Cargo dependencies if you
  need fully offline builds.
