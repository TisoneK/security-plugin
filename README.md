# security-plugin — RE analysis of `com.scorpio.securitycom`

A **reverse-engineering / security analysis** of a pre-installed Android
OEM app, produced with [REA](https://). This repository contains
**documentation and findings only** — no application binary and no
functional reconstruction.

## ⚠️ Disclaimer

- This is **independent security research for educational purposes**. It is
  **not affiliated with, endorsed by, or connected to** Transsion, Tecno,
  Infinix, itel, or the authors of the analyzed app.
- All product and company names are the property of their respective owners.
- The analyzed APK is **third-party proprietary software**. Its binary is
  **not redistributed here** (see [`.gitignore`](.gitignore)); only
  factual, statically-derived observations are documented.
- Analysis is **static only** — the app is never executed. Behavioral
  statements are **inferences** from the manifest, class/method names and
  bundled libraries, clearly marked as such and **not runtime-confirmed**.
- Provided "as is", with no warranty. Do not use any part of this to
  infringe copyright or to build or distribute silent-installer software.

## What the app is

`com.scorpio.securitycom` (v1.0.0.24) is a **Transsion-family (Tecno /
Infinix / itel) pre-installed "SecurityCom" component**. The evidence
(data models `RequestApkUrl` / `VersionControlBean`, `PullService`, the
OkDownload library, and `REQUEST_INSTALL_PACKAGES`) indicates it **pulls
APK download URLs from a backend and installs them**, with device-admin,
boot persistence, a keep-alive service, TEE-backed AES server
communication, a dialer secret-code entry point, and a signature-gated
exported IPC API. The DEX is ProGuard-obfuscated.

➡️ **Full write-up: [`analysis/ANALYSIS.md`](analysis/ANALYSIS.md)**

## Repository layout

| Path | Contents |
|------|----------|
| `analysis/ANALYSIS.md` | The reverse-engineering write-up (findings, risks). |
| `analysis/AndroidManifest.xml` | Decoded manifest (apktool), as evidence. |
| `analysis/classes_scorpio.txt` | REA class inventory of the app-owned classes. |
| `apk/` | **Not published** — the proprietary APK is kept locally only. |
| `.context_ledger/` | Persistent agent memory (Context Ledger protocol). |

## Reproducing the analysis

The analysis was produced with REA 6.7.0 + apktool 3.0.3 +
jadx-headless-mcp 0.7.1 on a JDK 17 (x64) toolchain. With a decoded APK
present locally:

```bash
source ~/.local/rea-toolchain/env.sh
APK="apk/<the>.apk"
rea decode-android-resources "$APK"
rea search-android-classes  "$APK" com.scorpio
rea inspect-android-class   "$APK" com.scorpio.service.SecurityComApiService
```

## Responsible use

If you are investigating pre-installed software on a device you own, that
is a legitimate use. Using this material to repackage, clone, or
redistribute the original app, or to build silent-install tooling, is
not — and is not supported here.
