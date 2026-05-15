# Installing Burp Suite's CA as an Android System Certificate

A step-by-step guide to trusting Burp's MITM certificate at the system level on an Android emulator. Required for intercepting HTTPS traffic from any app targeting Android 7+ (API 24), where user-installed CAs are ignored by default.

## Prerequisites

- An AVD running a **Google APIs** system image — **not** "Google Play Store." Play Store images ship as `user` (production) builds where `adb root` is denied and `/system` cannot be remounted. API 33 / Android 13 is the recommended target — API 34+ moved system CAs into APEX and requires a different procedure (see [Notes on Android 14+](#notes-on-android-14)).
- `openssl` on your host.
- Burp Suite with a proxy listener bound to an interface reachable from the emulator (e.g. all interfaces on port 8080).

Confirm the AVD is `userdebug`:

```bash
adb shell getprop ro.build.type
# Expected: userdebug
```

## 1. Export Burp's CA

In Burp:

**Proxy → Proxy settings → Tools / CA certificate → Export certificate in DER format**

Save as `burp_ca.der`.

## 2. Convert and rename to Android's expected format

Android looks up system CAs by `<subject_hash_old>.0`. Any other filename and the cert is invisible to the trust manager.

```bash
openssl x509 -inform DER -in burp_ca.der -out burp_ca.pem
HASH=$(openssl x509 -inform PEM -subject_hash_old -in burp_ca.pem | head -1)
cp burp_ca.pem "${HASH}.0"
echo "Cert ready: ${HASH}.0"
```

## 3. Boot the emulator with a writable system partition

```bash
emulator -avd <YourAVD> -writable-system -no-snapshot
```

`-writable-system` is **mandatory** and **per-boot** — it's not persisted in the AVD config. Forget it once and every later step fails.

## 4. Disable AVB verity

In a separate terminal:

```bash
adb wait-for-device
adb root
adb shell avbctl disable-verification
adb reboot
adb wait-for-device
adb root
```

`avbctl disable-verification` only takes effect after a reboot — the reboot is not optional.

## 5. Remount `/system` as read-write

```bash
adb remount
```

If it prints `Now reboot your device for settings to take effect`, do exactly that:

```bash
adb reboot && adb wait-for-device && adb root && adb remount
```

Confirm `/system` is writable before pushing:

```bash
adb shell mount | grep -E "/system\s"
```

Look for `rw` in the flags. If you still see `ro`, you missed either `-writable-system` at boot or the reboot after `disable-verification`.

## 6. Push the certificate

```bash
adb push ${HASH}.0 /system/etc/security/cacerts/
adb shell chmod 644 /system/etc/security/cacerts/${HASH}.0
adb shell chown root:root /system/etc/security/cacerts/${HASH}.0
adb reboot
```

## 7. Verify

On the host:

```bash
adb wait-for-device
adb shell ls /system/etc/security/cacerts/ | grep "${HASH}"
```

On the device:

**Settings → Security & privacy → More security settings → Encryption & credentials → Trusted credentials → System tab**

The Burp CA ("PortSwigger CA" or your custom CN) should appear here. If it shows up under the **User** tab instead, the push silently landed in the wrong location — recheck step 6.

## 8. Route the emulator's traffic through Burp

In the emulator:

**Settings → Network & internet → Internet → (long-press the active Wi-Fi) → Modify → Advanced → Proxy → Manual**

- Hostname: `10.0.2.2` (the host machine's alias from inside the emulator)
- Port: your Burp listener port (e.g. `8080`)
