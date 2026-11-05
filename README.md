# KernelSU-Next `boot.img` — OnePlus Nord N200 5G (`dre`)

Prebuilt, ABI-verified boot image with **KernelSU-Next** for
**LineageOS 24.0, build `20261030`** (`24.0-20261030-NIGHTLY-dre`, Android 17).

This repository contains **three files**:

| File | Purpose |
|---|---|
| `boot.img` | Flash this to the `boot` partition |
| `KernelSU_Next_v3.4.2-102-gb2c1ad00_33457-release.apk` | Matching KernelSU-Next manager (UAPI 6) |
| `README.md` | This guide |

> **Unofficial.** Not affiliated with OnePlus, LineageOS, or KernelSU-Next.
> Flashing a custom boot image can brick your device or destroy your data.
> Read the restore section before flashing. Use at your own risk.

---

## 1. Which LineageOS version is required

| Item | Required |
|---|---|
| Device | OnePlus Nord N200 5G — codename **`dre`**, SoC **SM4350 / holi** |
| ROM | **LineageOS 24.0, build `20261030`** |
| Build string | **`24.0-20261030-NIGHTLY-dre`** |
| Android version | **17** |
| Bootloader | **Unlocked** |
| Slot layout | A/B (flash to the active slot; the other slot may keep stock) |

Check before flashing:

```bash
adb shell getprop ro.lineage.version
# must print: 24.0-20261030-NIGHTLY-dre

adb shell getprop ro.build.version.release
# must print: 17
```

`ro.build.fingerprint` showing `OnePlusN200:12/SKQ1...` is **normal** — that is
the stock OxygenOS 12 vendor fingerprint, not a mismatch.

**Do not flash this on:**
- stock OxygenOS / OOS 12
- a different LineageOS build date
- a locked bootloader
- a different device (even another SM4350 phone)

The kernel vermagic and AVB properties are pinned to the 20261030 build.

---

## 2. What is inside `boot.img`

| Property | Value |
|---|---|
| Kernel | `5.4.302-qgki-g98a38783f42f` |
| Source | LineageOS `android_kernel_oneplus_sm4350`, branch `lineage-24.0`, commit `98a38783f42f` |
| KernelSU-Next | `legacy` nightly, `v3.4.0-legacy-24-gb933b1f7`, **Kprobes** hook mode |
| KernelSU UAPI | **6** (this is why the stable v3.4.2 manager is too old) |
| SCS / LTO / CFI | `SHADOW_CALL_STACK=y`, `LTO_CLANG=y` (ThinLTO), `CFI_CLANG=y`, `CFI_CLANG_SHADOW=y` |
| Boot header | v3, page size 4096, OS 17.0.0, patch 2026-10, empty cmdline |
| AVB | `SHA256_RSA4096`, AOSP test key, rollback index `1790812800` |
| Size | `100,663,296` bytes (exact `boot` partition size) |

### Checksums

```text
218ff9acd122752e6754547f48c5eb45af7c49676938f7c9e5a066469535bb0f  boot.img
aa43c1331ac1120b7be236fdcdc7af95b465ba1f8802216134774c32c4bc14c2  KernelSU_Next_v3.4.2-102-gb2c1ad00_33457-release.apk
```

Manager signing certificate SHA-256 (must match the kernel's expected key):

```text
79e590113c4c4c0c222978e413a5faa801666957b1212a328e46c00c69821bf7
```

Verify on your computer:

```bash
shasum -a 256 boot.img
# macOS/Linux; must match the value above

python3 - <<'PY'
# optional: check the APK's v2 signer certificate hash
import hashlib, struct

d = open('KernelSU_Next_v3.4.2-102-gb2c1ad00_33457-release.apk', 'rb').read()
eocd = d.rfind(b'PK\x05\x06')
cd = struct.unpack_from('<I', d, eocd + 16)[0]
assert d[cd - 16:cd] == b'APK Sig Block 42'

size = struct.unpack_from('<Q', d, cd - 24)[0]
pos, end = cd - size, cd - 24        # first id/value pair .. end of the pairs
v2 = None
while pos < end:
    n = struct.unpack_from('<Q', d, pos)[0]
    if struct.unpack_from('<I', d, pos + 8)[0] == 0x7109871a:
        v2 = d[pos + 12:pos + 8 + n]
        break
    pos += 8 + n

def lp(b, o):                           # read a uint32 length-prefixed slice
    n = struct.unpack_from('<I', b, o)[0]
    return b[o + 4:o + 4 + n], o + 4 + n

signers, _ = lp(v2, 0)
signer, _ = lp(signers, 0)
signed, _ = lp(signer, 0)
_, o = lp(signed, 0)                    # digests
certs, _ = lp(signed, o)
cert, _ = lp(certs, 0)
print(hashlib.sha256(cert).hexdigest())
PY
```

---

## 3. Before you flash

1. **Unlock the bootloader.** Unlocking wipes all user data. Do this first.
2. Install **Android platform-tools** (`adb`, `fastboot`).
3. Enable **Developer options → USB debugging**.
4. Confirm the phone is on the exact LineageOS build above.
5. Know how to restore stock (section 6). The other A/B slot is a fallback only
   if it has not already been updated.
6. Use a **USB data cable** and a reliable port.

---

## 4. Patch / flash the boot partition

### Important: this image is already patched

`boot.img` already contains the KernelSU-Next driver compiled into the kernel.
**You do not need to run a boot-image patcher, and there is no extra patch step.**
“Patching the boot file” on this device means writing the prepared image to the
`boot` partition with `fastboot`.

> The KernelSU-Next manager’s **“patch a file”** flow (LKM mode) is for GKI 2.0
> devices (Linux 5.10+). This phone is **GKI 1.0 / Linux 5.4**, so KernelSU-Next
> must be built into the kernel (`legacy` branch). A manager-patched image will
> not work here.

### Flash

```bash
# 1. Confirm the phone is connected and authorized
adb devices
# must show "device", not "unauthorized"

# 2. Reboot to the bootloader
adb reboot bootloader

# 3. Confirm fastboot sees it
fastboot devices

# 4. Note the active A/B slot (a or b)
fastboot getvar current-slot

# 5. Flash the image to the ACTIVE slot
fastboot flash boot boot.img

# 6. Reboot
fastboot reboot
```

`fastboot flash boot` writes to the active slot only. The other slot may keep
the stock image as a fallback. To write an explicit slot:

```bash
fastboot flash boot_a boot.img     # if current-slot is a
fastboot flash boot_b boot.img     # if current-slot is b
```

Do **not** flash `vendor_boot`, `dtbo`, or `vbmeta`. This image only replaces
the kernel in `boot`; the device tree and vendor modules stay stock.

### First boot

- First boot may take a few minutes.
- Verify the kernel:
  ```bash
  adb shell uname -r
  # 5.4.302-qgki-g98a38783f42f
  ```

---

## 5. Install the manager APK

Install the bundled **nightly** manager, not the stable v3.4.2 release:

```bash
adb install -r KernelSU_Next_v3.4.2-102-gb2c1ad00_33457-release.apk
```

Open **KernelSU Next**. It should show:

```text
Working
v3.4.0-legacy
33318-6
Kprobes
```

- `33318` is the KernelSU driver version.
- `6` is the **UAPI version** (the stable v3.4.2 manager reports 5).

### Why not the stable v3.4.2 APK?

The kernel reports UAPI **6**. The latest stable manager release (`v3.4.2`,
versionCode 33355) reports UAPI **5**, and shows:

> The current KernelSU-Next manager version 5 is too low for KernelSU-Next to
> work properly. Please upgrade manager to version 6 or higher!

The bundled APK is the official KernelSU-Next **nightly** build
(`v3.4.2-102-gb2c1ad00`, versionCode 33457) and reports UAPI **6**. It is signed
with the official KernelSU-Next key, so the kernel recognises it as the manager.

If the warning is still shown after installing:

```bash
adb shell am force-stop com.rifsxd.ksunext
```

then reboot once and reopen the manager (the kernel caches the manager APK
during boot).

---

## 6. Restore the stock boot image

Only the three files above are provided, so restore from the official OTA.

### Option A — extract `boot.img` from the exact OTA

Download the exact build:

```text
lineage-24.0-20261030-nightly-dre-signed.zip
sha256 4a8c8be4dd5b344fe90f445adfa2effd3c614b458549597ba1ec756ea494bcfd
```

From the LineageOS download mirror for `dre`. Then:

```bash
unzip -o lineage-24.0-20261030-nightly-dre-signed.zip payload.bin

# payload-dumper-go: https://github.com/ssut/payload-dumper-go
payload-dumper-go -p boot -o extracted payload.bin

adb reboot bootloader
fastboot flash boot extracted/boot.img
fastboot reboot
```

### Option B — sideload the full OTA (restores every partition)

Boot to **Lineage recovery** (Volume Down + Power → Recovery), then:

```bash
adb sideload lineage-24.0-20261030-nightly-dre-signed.zip
```

A normal OTA sideload preserves user data.

### Option C — switch to the other A/B slot

If the inactive slot still has the stock boot image:

```bash
fastboot getvar current-slot
fastboot --set-active=<the-other-slot>
fastboot reboot
```

This is not guaranteed after an OTA has updated both slots.

### If it bootloops

1. Hold **Volume Down + Power** until the bootloader appears.
2. Flash the stock `boot.img` from Option A, or sideload the OTA.
3. Do **not** re-lock the bootloader while a custom image is installed.

---

## 7. After a LineageOS OTA

A LineageOS OTA writes the inactive slot and switches to it. That overwrites the
kernel on the updated slot. After every OTA you must either:

- re-flash this `boot.img`, or
- rebuild a new image for the new LineageOS build (the kernel source revision
  changes between builds).

Do not flash this image onto a different LineageOS build date.

---

## 8. Verification (why this image is ABI-compatible)

The stock `vendor_dlkm` modules are built with Shadow Call Stack + ThinLTO + CFI
and must match the kernel ABI exactly. This image was checked against the **35
real modules extracted from the same 20261030 OTA**:

- **2,984 symbol-version CRCs matched — 0 mismatches, 0 missing symbols, 0 failing modules**
- Exact vermagic: `5.4.302-qgki-g98a38783f42f SMP preempt mod_unload modversions aarch64`
- `__cfi_slowpath` exported with the expected CRC `0x2cd90edd`
- Cross-DSO CFI type IDs intersect on 131 (`wlan.ko`), 65 (`camera.ko`) and 12 (`machine_dlkm.ko`) shared IDs
- DTB byte-identical to the stock `vendor_boot` DTB
- AVB footer verified with `avbtool verify_image`
- KernelSU-Next: 367 `ksu_*` symbols in `System.map`

**Boot-tested on the device.** After flashing, the phone booted with the custom
kernel and the manager was granted root:

```text
adb shell uname -r
# 5.4.302-qgki-g98a38783f42f

adb shell getprop ro.lineage.version
# 24.0-20261030-NIGHTLY-dre
```

```text
KernelSU: install fd for manager: 10243
KernelSU: allow root for: 10243
```

The manager's home screen could not be screenshotted during the reference run
because the phone was locked with a PIN; the kernel log and the installed
`versionCode=33457` were used to confirm it instead. Keep the restore
instructions above anyway.

---

## 9. Source and licenses

- Linux kernel: [LineageOS/android_kernel_oneplus_sm4350](https://github.com/LineageOS/android_kernel_oneplus_sm4350) — **GPL-2.0**
  - branch `lineage-24.0`, commit `98a38783f42f`
- [KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next) — **GPL-2.0**
  - kernel driver: `legacy` branch, `b933b1f7`
  - manager APK: official nightly `v3.4.2-102-gb2c1ad00` (versionCode 33457)
  - nightly download: <https://nightly.link/KernelSU-Next/KernelSU-Next/workflows/build-manager-ci/dev/manager>
- AOSP test key used for AVB (public, not a secret).

The prebuilt kernel is a GPL-2.0 derivative. The complete corresponding source
is the LineageOS commit above plus the KernelSU-Next `legacy` commit above. The
local build workspace in this project contains the full patches, config, build
log and reproduction guide; it is not part of this three-file upload.

**No warranty.** The software and images are provided “as is”, without warranty
of any kind. You are responsible for your own device.
