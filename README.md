# frida-android-audit

A Frida-based toolkit for runtime analysis of Android applications. Bundles a comprehensive instrumentation script and the emulator setup procedure needed to run it end-to-end against modern apps.

The script bypasses three categories of app-side defenses:

- **Emulator detection** — Build property spoofing, QEMU pipe checks, native CPU family detection.
- **Root detection** — Filesystem checks, package manager queries, shell command interception, system property reads. Both Java and native layers.
- **SSL certificate pinning** — 20+ targeted hooks plus a generic `SSLPeerUnverifiedException` auto-patcher.

## Disclaimer

This toolkit is for **authorized security testing, education, and research only**. Use it against:

- Apps you own.
- Apps you have explicit written authorization from the rights holder to test.
- CTF challenges and intentionally vulnerable training apps.

Bypassing security controls on apps you don't own or aren't authorized to test may violate the Computer Fraud and Abuse Act (US), the Computer Misuse Act (UK), and equivalent laws elsewhere. You are responsible for ensuring your use is lawful.

## Prerequisites

- macOS or Linux host (Windows works under WSL; commands below assume a POSIX shell).
- [Android SDK](https://developer.android.com/studio) with `emulator`, `adb`, `sdkmanager`, `avdmanager` on `PATH`.
- [Frida](https://frida.re/) — `pip install frida-tools`.
- An MITM proxy. This guide assumes [Burp Suite](https://portswigger.net/burp); adapt the CA export step for mitmproxy, Charles, or HTTP Toolkit.
- An AVD on a **Google APIs** system image — *not* Google Play Store. The Play Store image blocks `adb root` and makes the system CA install impossible. API 33 / Android 13 is the recommended target.

## Quick start

### 1. Trust your proxy's CA at the system level

Without this, WebView traffic and any app that ignores user CAs will refuse to talk to your proxy regardless of how thorough your Frida hooks are. See [`docs/install-burp-ca.md`](docs/install-burp-ca.md) for the full procedure.

### 2. Start frida-server on the emulator

```bash
# Match the frida-server architecture to your AVD (x86_64 for typical x86_64 AVDs)
adb push frida-server /data/local/tmp/frida-server
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server &"
```

### 3. Spawn the target with the script attached

```bash
frida -U -f <target.package.name> -l frida_bypass.js
```

## What the script bypasses

### Emulator detection

- Overwrites `android.os.Build` fields (`PRODUCT`, `MANUFACTURER`, `BRAND`, `DEVICE`, `MODEL`, `HARDWARE`, `FINGERPRINT`) to impersonate a Samsung Galaxy Note 7.
- Hooks `java.io.File.exists()` to deny existence of QEMU-specific pipes (`qemud`, `qemu_pipe`).
- Intercepts the native `android_getCpuFamily` to return `ARM64` instead of `x86`/`x86_64`.

### Root detection

- **Filesystem**: Java `UnixFileSystem.checkAccess` and native libc `fopen`/`access` hooks for 36 known root binary paths and `magisk`-named files.
- **Package manager**: `getPackageInfo` returns "not found" for 26 known root / Xposed / Magisk packages.
- **Shell**: `Runtime.exec` (6 overloads), `ProcessBuilder.start`, and `ProcessImpl.start` hooks for `su`, `which su`, `mount`, `getprop`, `id`, `build.prop`.
- **System properties**: `SystemProperties.get` and native `__system_property_get` return safe values for `ro.build.tags`, `ro.secure`, `ro.debuggable`, `service.adb.root`, `ro.build.fingerprint`.
- **Native `system()`**: libc `system()` calls for the same patterns above.

### SSL certificate pinning

Targeted hooks for:

- `HttpsURLConnection` (default hostname verifier and SSL socket factory)
- `SSLContext.init` (custom no-op `X509TrustManager`)
- `TrustManagerImpl` — `checkTrustedRecursive` and `verifyChain`, which is what defeats Network Security Config pinning on Android 7+
- OkHttp v3 (`CertificatePinner.check` × 4 overloads, including `check$okhttp`)
- Trustkit (`OkHostnameVerifier`, `PinningTrustManager`)
- Conscrypt `OpenSSLSocketImpl`, `OpenSSLEngineSocketImpl`, Apache Harmony `OpenSSLSocketImpl`
- `CertPinManager` (Conscrypt + CWAC-Netsecurity)
- Appcelerator, Cordova, Boye, Netty, PhoneGap, Squareup OkHttp (pre-v3)
- IBM WorkLight / MobileFirst (full quadruple bypass)
- Appmattus Certificate Transparency — interceptor, trust manager, and hostname verifier variants
- `WebViewClient.onReceivedSslError` with `handler.proceed()`

Plus a generic `SSLPeerUnverifiedException` auto-patcher that dynamically null-implements any throwing method on first failure — catches unknown or obfuscated pinning libraries.

## Tested against

[`httptoolkit/android-ssl-pinning-demo`](https://github.com/httptoolkit/android-ssl-pinning-demo) — 12/12 SSL pinning challenges pass, including:

- Unpinned baseline (HTTP/1, HTTP/3, WebView)
- Network Security Config-pinned requests
- SSLContext-pinned requests
- OkHttp, Volley, Trustkit
- Appmattus Certificate Transparency (interceptor, OkHttp combo, raw TLS, WebView)

WebView tests require the system CA install — see prerequisites.

## Acknowledgements

The SSL pinning section originated from publicly-circulated bypass scripts, notably HTTP Toolkit's [frida-android-unpinning](https://github.com/httptoolkit/frida-android-unpinning) and akabe1's [frida-multiple-unpinning](https://codeshare.frida.re/@akabe1/frida-multiple-unpinning/) on Frida CodeShare. The root-detection section draws from multiple community sources.

## Contributing

PRs welcome. When adding a new library hook, please include:

- The library's public repo or homepage URL
- A note on which version(s) you tested against
- A `console.log("[+] ...")` / `console.log("[ ] ...")` pair so users can tell at a glance whether the hook attached.

## References

- [Frida documentation](https://frida.re/docs/home/)
- [OWASP Mobile Application Security Testing Guide](https://mas.owasp.org/MASTG/)
- [HTTP Toolkit — Android 14 System CA install](https://httptoolkit.com/blog/android-14-install-system-ca-certificate/)
- [Android Network Security Configuration](https://developer.android.com/privacy-and-security/security-config)
