# PO Pixel 1.3.0 — standalone rebuild

`POPixel-v1.3.0.apk` is **Pocket Operator for Pixel 1.3.0** (the real app, unmodified content)
with the Google Play **PAIRIP** protection layer removed, rebuilt so it installs and runs on
**any arm64 Android device — no root, no Pixel, no Play Store licence**.

| | |
|---|---|
| Package | `com.teenageengineering.pocketoperatorforpixel` |
| Version | 1.3.0 (`versionCode` 39) |
| Label | PO Pixel |
| Min / target SDK | 30 (Android 11) / 35 |
| ABI | `arm64-v8a` only |
| Download size | 109 MB (≈235 MB installed) |
| SHA-256 | `a109b407f034e355e52e5d61bd7762a54fc43978c4e2c5787eae83a737144860` |
| Signing | `CN=PO Debug` (self-signed, see caveats) |

---

## Changelog

* v1 (2026-11-10): initial version
* v2 (2026-11-11): fixed video import

---

## Install

1. Download [POPixel-v1.3.0.apk](https://github.com/chubiduwa/pocket-operator-for-non-pixel-too/releases/download/v1/POPixel-v1.3.0.apk)
2. **Uninstall the Play Store version first** (if present). This build keeps the original package
   name but is signed with a different key, so Android will refuse to update over it.
3. Copy the APK to the device and open it, or:
   ```
   adb install POPixel-v1.3.0.apk
   ```
4. Launch **PO Pixel**. On first run it unpacks its assets (song, tutorial videos, ~45 MB of
   audio-engine and Unity data) — the splash screen can sit for 20–40 s before the tutorial appears.

Requirements: Android 11+, a 64-bit ARM CPU, OpenGL ES 2.0, and roughly 400 MB free storage
(109 MB APK plus ~235 MB installed).

A camera is *not* required (it is only used to import video).

---

## Why a rebuild was needed

The Play release is wrapped in **PAIRIP**:

* `libpairipcore.so` — a VM interpreter that decrypts and patches code at startup and verifies the
  APK's signing certificate.
* Encrypted blobs in `assets/` carrying the DEX string literals; the Java layer calls into the VM
  during `Application.<clinit>`.
* Three libraries (`libunity.so`, `libil2cpp.so`, `libavcodec.so`) ship with an encrypted prefix of
  `.text`, rewritten `.data`, a `DT_INIT` pointing at a PAIRIP stub, and a subset of imports whose
  `.dynsym`/`.rela.plt` entries were deleted.
* Device gating: a required `PIXEL_2020_EXPERIENCE` feature, a Java check that calls `finish()` on
  non-Pixel hardware, and a Play licence check that exits the process when the account does not own
  the app.

Any modified or re-signed APK that still contained PAIRIP aborted with `std::bad_alloc` inside
`executeVM`. So the protection was removed rather than bypassed.

## What was changed

**Java layer** — the PAIRIP classes are gone entirely. The app runs on a small reimplementation of
`PluginEntry` (file handling, permissions, camera cut-out, external files dir) plus the stock Unity
`UnityPlayerActivity`. Consequences: no licence check, no VM startup program, and
`RaccoonGetPixelVersion()` returns `6` so the "requires a Pixel" `finish()` never fires.

**Native libraries** — the three protected libraries were reconstructed from the *original*
process by snapshotting them in a well-defined state (just after PAIRIP's VM had done its work but
before the library initialised itself), then rebuilding `.text`/`.data` and replacing the PAIRIP
entry point with a small loader shim that:

* resolves the ~190 (`libunity`) / 101 (`libavcodec`) imports whose symbols PAIRIP deleted, via
  `dlsym()`, and writes them into the GOT;
* applies the `.data` pointer fixups that PAIRIP's VM used to do;
* installs bdwgc's signal handlers' expected state so `il2cpp`'s GC starts cleanly.

Everything else — `global-metadata.dat`, `data.unity3d`, videos, the raccoon audio engine — is
byte-identical to the shipped app.

**Manifest** — the `PIXEL_2020_EXPERIENCE` requirement is gone, `android.hardware.camera` is no
longer required, the label is `PO Pixel`, and the activity is pinned to `sensorLandscape`
(the game still sets its own orientation at runtime when it picks a layout).

## Verified

Only tested on:

* OnePlus 7 Pro (Android 12)
* Pixel Tablet (Android 15)

Exercised: cold boot from a clean install, first-run tutorial, menu → main UI, transport, track
selection, audio output (OpenSL ES + AAudio), project persistence, unity/il2cpp/ffmpeg libraries
loaded.

## Caveats

* **Self-signed** (`CN=PO Debug`). Install requires "unknown sources"; you cannot update over the
  Play build. Google Play services will not manage or update it.
* **Not tested** on 16 KB-page devices, x86/x86_64 devices, or Android versions below 11.
* On phones the game selects its internal **phone layout**, which is portrait-oriented and denser
  than the tablet layout; that is the app's own behaviour, not a rebuild artefact.
* Video import (camera / file picker) and export were not deeply exercised.

## Safety

Check the download with:

```
shasum -a 256 POPixel-v1.3.0.apk
# a109b407f034e355e52e5d61bd7762a54fc43978c4e2c5787eae83a737144860
```

---

## Trademarks

**Pocket Operator** is a trademark of Teenage Engineering AB. **Pixel** and **Google Play** are
trademarks of Google LLC. All other product names, logos and artwork referenced here belong to
their respective owners.

This is an unofficial, community-made rebuild. It is not produced, sponsored, endorsed by or
affiliated with Teenage Engineering or Google, and no ownership of the original application,
its name, its icons or its artwork is claimed or granted by it. Those marks and assets are used
only to identify the software this build is derived from.
