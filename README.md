# OneLineage

Hot-switch between stock **One UI** and a **EvoX 11.8 (Android 16) GSI** on a rooted Samsung phone, with no reflashing and no second partition. OneLineage stages the GSI under `/data/local/tmp`, pivots into it as a container with its own zygote and system_server, and can hand the screen back to One UI.

If something isn't working and you have a fix feel free to make a pr!

> ## Disclaimer
> **This has been tested on only a handful of devices, and it seems if you want to install it without making modifications, it has to be on Android 16**
> It may not work on other Samsung models, other One UI versions, or other devices at all. It is experimental, it replaces core parts of a running system, and it can crash, hang or soft-brick your phone, fill `/data`, or break radio/SMS. **Back up everything first. Use at your own risk.**
> Also, I did use AI a fair bit in this project, it would have taken a LOT longer without it.

## Requirements
- A rooted Samsung phone (the tested one is an S26 Ultra on One UI 8.5)
- Enough free space on `/data` for the extracted GSI
- Internet access for the GSI download (EvoX 11.8 vanilla arm64 A/B GSI image)

## What's in here
| Path | What it is |
| --- | --- |
| `src/`, `res/`, `AndroidManifest.xml` | The Android app (`com.wewaffles.onelineage`) |
| `assets/master_prepare.sh` | Stages the rootfs, applies Lineage-specific compatibility overlays only to the matching image, fixes labels/ownership |
| `assets/master_inner.sh` | Runs inside the container, starts the guest zygote |
| `assets/switch_to_lineage.sh` / `return_to_oneui.sh` | Switch in and back out |
| `assets/zygote_exec` | Static aarch64 launcher for the guest zygote |
| `build_apk.sh` | Builds and signs the APK |

## EvoX compatibility

The app offers the EvoX 11.8 vanilla and GApps arm64 A/B images. Vanilla uses a `system.7z` archive hosted on GoFile; GApps still downloads the `.img.xz` release from [Doze-off/EvoX_treble](https://github.com/Doze-off/EvoX_treble/releases/tag/2026-06-26). The app extracts the image and stages it as `system.raw`. The inspected vanilla image reports Android 16 / SDK 36 and build ID `BP4A.251205.006`. Samsung's PermissionController references `android.permission.RANGING`, which this EvoX platform does not define. The bundled patched PermissionController removes that mapping, so it is applied to EvoX too. EvoX's own `services.jar` already contains Samsung proximity fallback and brightness scaling; those behaviors remain intact. The EvoX-specific `services_evox_patched.jar` is based on EvoX's JAR and adds the Samsung lights AIDL path with the original patch's sysfs max-brightness detection. Original Lineage assets remain separate. The EvoX service also selectively ports Samsung fingerprint/sysinput HAL discovery, under-display fingerprint mask-brightness observation, and UDFPS metadata helpers while leaving EvoX sensor registration intact. This service change has been rebuilt and checked for DEX 039 layout and duplicate class definitions; device boot and availability of the host Samsung vendor HAL services remain unverified. The Samsung RAW-image framework changes in the donor (`ImageDecoder.isMimeTypeSupported`, `ExifInterface.isSupportedMimeType`, and the `MediaFile` registration) are already bytecode-identical in EvoX 11.8, so EvoX keeps its own framework JAR without a redundant overlay.

## Patches

### Patched system files (LineageOS profile only; EvoX keeps its own matching framework files)
- `services_patched.jar` over `system/framework/services.jar`
- `framework_patched.jar` over `system/framework/framework.jar`
- `PermissionController_patched.apk`: removes Samsung PermissionController's `android.permission.RANGING` mapping, which the EvoX platform does not define; this patch is applied to EvoX as well
- Optional bind-mounts when present: `framework-res_patched.apk`, `libaudio_aidl_conversion_common_cpp_patched.so`, `libaudioclient_aidl_conversion_patched.so`

### Boot and environment fixes
1. **APEX paths in the container.** Host `/system/apex` and `/system_ext/apex` are bound into the rootfs after the GSI payloads are extracted, so PackageManagerService stops dying in `scanApexPackages`.
2. **SELinux labels on the staged `/data`.** The tree is relabeled like a real `/data` (`system_data_file`, per-package `app_data_file`) instead of `shell_data_file`. Also `setenforce 0` during the takeover and `setenforce 1` on return.
3. **Audio hang.** Host `audioserver`/`cameraserver` are left running (they used to be stopped/frozen, which deadlocked `AudioService`). Only the host zygote is suppressed.
4. **Runaway `sf.log`.** SurfaceFlinger output goes to `/dev/null` (it was growing to hundreds of MB in minutes).
5. **App data ownership.** `fix_app_dirs.sh` re-owns app data dirs to the uid PackageManager assigned, strips write bits from dex/apk/jar files (ART refuses writable dex), and creates missing dirs such as `/data/misc/bluedroid`.
6. **No installd.** `ensure_app_storage.sh` plus a 15 s sweeper create `/data/user/0`, `/data/user_de/0` and shared-storage dirs for new installs, so apps installed inside Lineage can launch.
7. **Gesture navigation.** Overlay idmaps are generated with `idmap2 --ignore-overlayable` before zygote starts.
8. **Bluetooth crash loop** fixed by creating `/data/misc/bluedroid` before boot.
9. **adb.** `Settings.Global.ADB_ENABLED` is pre-seeded, host `adb_keys` are copied into the guest, and `adb_watchdog.sh` keeps adbd and the USB gadget alive.
10. **Networking.** The preferred APN is switched to `IPV4V6`, and `net_fix.sh` maintains a private routing table (4242) because the guest's ConnectivityService can't program the host's netd.
11. **SMS.** `ims_watchdog.sh` keeps One UI's `imsd` running and `switch_to_lineage.sh` waits for voice registration before pivoting.
12. **Boot diagnostics.** Each boot writes `/data/boot_failures.log`.

## Known issues
- Returning to One UI may still need a manual reboot.
- The guest binds the host's `media.codec`, whose AIDL hash differs from the GSI's, which can cause high CPU during codec enumeration.
- No IMS stack in the guest, so **voice calls don't work** (data and SMS depend on One UI's `imsd`).
- Developer options crash (the guest registers no `adb` service).
- DTMF tones glitch, and deep-buffer audio hangs, so it stays disabled.
- Never kill the host `netd` while Lineage is running.

## Building
The `Build APK` GitHub Actions workflow builds and debug-signs the app on pushes, pull requests, or when started manually. Download `OneLineage-<commit>` from the run's artifacts to install it for testing. The generated debug keystore is only for test builds; it will not match a separately signed release build.

To build locally, install JDK 17 and Android SDK platform 34 / build-tools 34.0.0, set `ANDROID_SDK_ROOT` (or `ANDROID_HOME`), then run `bash ./build_apk.sh`. The APK is written to `build/OneLineage.apk` by default. Set `OUT_APK` or `KEYSTORE` to override the output path or signing keystore.

## Credits
EvoX GSI by Doze-off. EvoX and Android are trademarks of their respective owners. Not affiliated with Samsung or Google.
