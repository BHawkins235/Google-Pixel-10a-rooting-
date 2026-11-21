# Rooting a Pixel 10a (stallion) with APatch: Steps and Troubleshooting

All commands are for **Windows PowerShell**, run from your platform-tools folder. Replace file names with the ones in your own factory image.

**Build used:** `stallion_beta-cp31.260623.012` (Android 17 beta, bootloader 17.0)
**Root method:** APatch (patched `boot.img`, no Magisk)

> **Key lesson:** Wi-Fi and Bluetooth broke because partitions from different builds were mixed. Flashing the bootloader, radio, and images all from the **same factory zip** fixed it. Never mix files from different builds or models.

---

##  Warnings

- Unlocking the bootloader **wipes the phone**. Back up first.
- Rooting usually voids the warranty and can break banking apps and Google Wallet.
- A beta bootloader can trigger **anti-rollback protection**, which can block flashing older bootloaders later.
- Use files from **one** factory zip that matches your exact model (`stallion` = Pixel 10a).
- Keep the extracted factory folder as your recovery kit, outside OneDrive (for example `C:\factory`).

---

## 1. Setup

1. Install the latest **Android SDK Platform-Tools** from Google and unzip them to `C:\platform-tools`.
2. Install Google's USB driver if Windows doesn't detect the phone.
3. On the phone: Settings > About phone > tap **Build number** 7 times.
4. Settings > System > Developer options: turn on **USB debugging** and **OEM unlocking**.
5. Open PowerShell in the platform-tools folder:

```powershell
cd C:\platform-tools
.\adb devices
```

Accept the USB debugging prompt on the phone. The device should be listed.

---

## 2. Unlock the bootloader (wipes the phone)

```powershell
.\adb reboot bootloader
.\fastboot flashing unlock
```

Confirm on the phone with the volume and power buttons. After the wipe, redo setup and re-enable USB debugging.

Check it worked:

```powershell
.\fastboot getvar unlocked
.\fastboot getvar product
```

`unlocked` should be `yes` and `product` should be `stallion`.

---

## 3. Get the factory image and extract boot.img

1. Download the factory image for your **exact model and build** from Google's factory images page.
2. Copy the zip to `C:\factory` (not OneDrive, which can lock files).
3. Extract:

```powershell
cd C:\factory
Expand-Archive .\stallion_beta-*-factory-*.zip -DestinationPath C:\factory
cd C:\factory\stallion_beta-cp31.260623.012
Expand-Archive .\image-stallion_beta-*.zip -DestinationPath .\image
Get-ChildItem .\image\boot.img
Get-ChildItem .\*.img
```

You should see:

- `bootloader-...img` and `radio-...img` in the factory folder
- `boot.img`, `vendor_boot.img`, `vendor_dlkm.img`, `vendor.img` and others in `.\image`

Use `boot.img`, not `init_boot.img`, for APatch.

Confirm the phone's current build matches the zip:

```powershell
.\adb shell getprop ro.build.display.id
```

---

## 4. Get a clean, matching stock state first

Before rooting, make sure Wi-Fi and Bluetooth work on stock. The simplest way is Google's own script, unedited, with the phone in the bootloader:

```powershell
.\adb reboot bootloader
cd C:\factory\stallion_beta-cp31.260623.012
.\flash-all.bat
```

This **wipes the phone**. To keep data, open `flash-all.bat` in Notepad and remove only the `-w` flag.

Boot without root and confirm Wi-Fi and Bluetooth work.

### Manual flash order (alternative to flash-all)

The syntax is always `fastboot flash <partition> <file>`.

```powershell
.\adb reboot bootloader
.\fastboot flash bootloader .\bootloader-stallion_beta-stallion-17.0-15819938.img
.\fastboot reboot bootloader
.\fastboot flash radio .\radio-stallion_beta-g5400i-260604-260807-b-16035863.img
.\fastboot reboot bootloader
.\fastboot flash boot .\image\boot.img
.\fastboot flash vendor_boot .\image\vendor_boot.img
.\fastboot reboot fastboot
.\fastboot flash vendor_dlkm .\image\vendor_dlkm.img
.\fastboot flash vendor .\image\vendor.img
.\fastboot reboot
```

Use your real file names. To list them: `Get-ChildItem .\*.img`.

---

## 5. Install APatch and patch boot.img

1. Download the latest APatch Manager `.apk` from the official GitHub releases page (`github.com/bmax121/APatch`) and install it on the phone.
2. Push the stock `boot.img` from your matching zip:

```powershell
.\adb push .\image\boot.img /sdcard/Download/boot.img
```

3. In APatch tap **Install**, choose **Select a boot image to patch**, pick `boot.img`, and set a strong **SuperKey**. Remember it.
4. Pull the patched file to the PC (use the real name the app creates):

```powershell
.\adb pull /sdcard/Download/apatch_patched_xxxxx.img .
```

---

## 6. Test the patched image without flashing

```powershell
.\adb reboot bootloader
.\fastboot boot .\apatch_patched_xxxxx.img
```

This boots the patched image once without writing it. A normal reboot returns to the stock boot image.

After it boots:

1. Open APatch and enter your SuperKey. It should show **Working**.
2. Turn on Wi-Fi and Bluetooth and confirm they work.
3. Check root:

```powershell
.\adb shell su -c id
```

You should see `uid=0(root)`.

---

## 7. Flash the patched image permanently

Only do this if Wi-Fi, Bluetooth, and root all worked in the test.

```powershell
.\adb reboot bootloader
.\fastboot flash boot .\apatch_patched_xxxxx.img
.\fastboot reboot
```

Verify again after the reboot:

```powershell
.\adb shell
su
id
```

---

## 8. Restore stock boot.img (remove root)

```powershell
.\adb reboot bootloader
.\fastboot flash boot .\image\boot.img
.\fastboot reboot
```

Flash both slots if needed:

```powershell
.\fastboot getvar current-slot
.\fastboot --slot=all flash boot .\image\boot.img
```

---

## Troubleshooting

### A. `unknown partition '...img'` / `cannot determine image filename`

**Cause:** The partition name is missing, so fastboot treats the file path as the partition. Other causes are a line break splitting the command, copy/paste smart quotes, or an empty variable.

**Fix:** Use `fastboot flash <partition> <file>`.

```powershell
.\fastboot flash bootloader .\bootloader-stallion_beta-stallion-17.0-15819938.img
```

Print variables before using them and confirm the file exists:

```powershell
$bl = (Get-ChildItem .\bootloader-*.img).Name
$bl
Get-ChildItem .\*.img | Select-Object Name
```

### B. `<waiting for any device>` (fastboot or flash-all.bat)

**Cause:** The PC can't see the phone in fastboot mode.

**Fix, in order:**

```powershell
.\adb reboot bootloader
.\fastboot devices
```

1. Make sure the phone is on the **bootloader screen**. If adb can't reach it, hold **Power + Volume Down**.
2. Check Device Manager. If there is a warning icon, update the driver: right-click > Update driver > Browse > Let me pick > Android Device > **Android Bootloader Interface**, or point it at Google's `usb_driver` folder.
3. Use a data-capable USB cable and a rear USB port on the PC, preferably USB 2.0. Avoid hubs.
4. Use the latest Platform-Tools and put them on PATH so `flash-all.bat` finds fastboot:

```powershell
$env:Path = "C:\platform-tools;" + $env:Path
Get-Command fastboot
```

5. Close Android Studio and anything else using adb, then run `.\adb kill-server`.
6. Run PowerShell as Administrator, or reboot the PC.
7. As an alternative that avoids drivers, use Android Flash Tool (`flash.android.com`) in Chrome.

### C. Wi-Fi shows "Networks unavailable" or "Wi-Fi is disabled", Bluetooth dead

**Cause (what happened here):** Mismatched partitions. A beta bootloader or radio was combined with vendor, boot, or kernel files from a different build. The Wi-Fi and Bluetooth drivers and firmware live in the bootloader and vendor partitions, not the radio (the radio is the cellular modem). A patched `boot.img` or a root module from another build can break them too.

**Diagnose:**

```powershell
.\adb shell getprop ro.build.display.id
.\adb shell getprop ro.bootloader
.\adb logcat -d | Select-String -Pattern "wlan|wifi|bluetooth|firmware|dlkm|Unknown symbol" -CaseSensitive:$false
.\adb shell svc wifi enable
.\adb shell cmd wifi status
```

How to read the logs:

- `firmware not found` or `failed to load`: vendor partition wrong or damaged.
- `Unknown symbol` or module errors: kernel and vendor modules don't match.
- No Wi-Fi driver lines at all: driver never loaded.

**Fix (this resolved it):** Reflash the bootloader, radio, and images **all from the same factory zip**. Easiest is the unedited `flash-all.bat`. Or use the manual order in section 4.

**Other things to try:**

```powershell
# Remove root and use stock boot
.\adb reboot bootloader
.\fastboot flash boot .\image\boot.img
.\fastboot flash vendor_boot .\image\vendor_boot.img
.\fastboot reboot

# Flash both slots
.\fastboot getvar current-slot
.\fastboot --slot=all flash boot .\image\boot.img
.\fastboot --slot=all flash vendor_boot .\image\vendor_boot.img

# Reflash vendor partitions
.\adb reboot bootloader
.\fastboot reboot fastboot
.\fastboot flash vendor_dlkm .\image\vendor_dlkm.img
.\fastboot flash vendor .\image\vendor.img
.\fastboot reboot

# Airplane mode block
.\adb shell settings put global airplane_mode_on 0
.\adb shell cmd connectivity airplane-mode disable
.\adb reboot
```

Also try Settings > System > Reset options > **Reset Wi-Fi, mobile & Bluetooth**.

### D. `/system/bin/sh: wifi: inaccessible or not found`

**Cause:** `wifi` is not a command. It is an argument to `svc` or `cmd`.

**Fix:**

```powershell
.\adb shell svc wifi enable
.\adb shell cmd wifi status
.\adb shell cmd wifi set-wifi-enabled enabled
```

### E. `/system/bin/sh: sudo: inaccessible or not found`

**Cause:** Android has no `sudo`. Use `su`.

**Fix:**

```powershell
.\adb shell
su
id
```

Or in one line:

```powershell
.\adb shell su -c id
```

### F. `su` denied or not found

1. Open APatch, enter your SuperKey, and confirm the home screen says **Working**. If it says "Not installed", the patched boot image isn't running. Boot or flash it again.
2. Open the **SuperUser** tab and grant **Shell** (`com.android.shell`) access, then retry.
3. Approve any root prompt on the phone quickly. It times out.
4. Remove other root methods. Use one of APatch, Magisk, or KernelSU only.
5. Run `su` inside `adb shell`, or as `.\adb shell su -c "id"`.

### G. Bootloop after flashing

1. Hold **Power + Volume Down** to reach the bootloader.
2. Flash the stock boot image back:

```powershell
.\fastboot flash boot .\image\boot.img
.\fastboot reboot
```

3. If that fails, run the full unedited `flash-all.bat`. It wipes the phone unless you remove `-w`.

### H. Files on OneDrive misbehave

Files on the Desktop or other synced folders can be placeholders or get locked during a flash. Move the extracted factory folder to a plain location like `C:\factory` and run commands from there.

---

## Habits that keep Wi-Fi and root working

- Flash only complete sets from **one** factory zip. Never mix builds or models.
- Don't flash the bootloader or radio on their own.
- Confirm every file name contains `stallion` and that `fastboot getvar product` returns `stallion`.
- Always test with `fastboot boot` before flashing a patched image.
- Re-patch the new `boot.img` after every system update. Don't reuse an old patched image.
- Add root modules one at a time and reboot after each.
- Keep the stock `boot.img` and `vendor_boot.img` on your PC for recovery.
- Check APatch's supported kernel range. Reports say it supports kernels up to 6.1, so a newer beta kernel may not work.
- If APatch won't work on your build, fall back to Magisk, or to the stable factory image.

---

## Quick command reference

| Task | Command |
|---|---|
| List adb devices | `.\adb devices` |
| Reboot to bootloader | `.\adb reboot bootloader` |
| List fastboot devices | `.\fastboot devices` |
| Check model | `.\fastboot getvar product` |
| Check bootloader unlocked | `.\fastboot getvar unlocked` |
| Check active slot | `.\fastboot getvar current-slot` |
| Check build | `.\adb shell getprop ro.build.display.id` |
| Test patched boot | `.\fastboot boot .\apatch_patched_xxxxx.img` |
| Flash patched boot | `.\fastboot flash boot .\apatch_patched_xxxxx.img` |
| Flash stock boot | `.\fastboot flash boot .\image\boot.img` |
| Check root | `.\adb shell su -c id` |
| Wi-Fi logs | `.\adb logcat -d \| Select-String -Pattern "wlan\|wifi\|firmware" -CaseSensitive:$false` |
| Reboot | `.\fastboot reboot` |

## Links
Google pixel 10a image https://developer.android.com/about/versions/17/qpr1/download/ |
