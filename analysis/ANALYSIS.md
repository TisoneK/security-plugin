# Reverse-Engineering Analysis — `com.scorpio.securitycom`

> Static analysis only. The APK is **never executed**. Behavioral claims
> below are **inferences** from the manifest, class/field/method names and
> bundled libraries — not confirmed runtime behavior. Code is ProGuard-
> minified, so most internal method names are single letters (`a`, `b`, …).

## Subject

| Field | Value |
|-------|-------|
| File | `com.scorpio.securitycom_1.0.0.24-32_minAPI28(nodpi)_apkmirror.com.apk` |
| SHA-256 | `35ec8526d518ef0a0f8d1514b99827925fde12e1396fd0b14f0965badfa14fef` |
| Package | `com.scorpio.securitycom` |
| Version | `1.0.0.24` (code `32`) |
| minSdk / targetSdk | 28 / 29 |
| Source | apkmirror.com (third-party redistribution) |
| Toolchain | REA 6.7.0 · apktool 3.0.3 · jadx-headless-mcp 0.7.1 (JDK 17 x64) |
| DEX | single `classes.dex` (~1.79 MB), 754 classes total, 24 app-owned |

## What this app is

Metadata ties it to **Transsion** (the OEM behind Tecno / Infinix / itel
budget Android phones):

```
<meta-data android:name="tran_athena_version"   android:value="2.2.0.1"/>
<meta-data android:name="tran_coreutil_version" android:value="1.0.61.04"/>
```

Combined with the component set below, this is a **pre-installed OEM
system component** — a "Security" / device-management app that **pulls
APK download URLs from a backend and installs them**, with persistence,
device-admin, and TEE-backed encrypted server communication. This matches
the preinstalled "app push / silent installer" pattern that has been
documented in security research on low-cost Android handsets.

## Permissions (from manifest)

| Permission | Why it matters |
|------------|----------------|
| `INTERNET`, `ACCESS_NETWORK_STATE`, `ACCESS_WIFI_STATE` | Server comms |
| `REQUEST_INSTALL_PACKAGES` | **Can install other APKs** |
| `RECEIVE_BOOT_COMPLETED` | **Autostart on boot** (persistence) |
| `WRITE_/READ_EXTERNAL_STORAGE` | Stage downloaded APKs |
| `com.android.settings.SETTINGS_CONTENT_PROVIDER` | Read device settings |
| `com.securitycom.permission.ACCESS` (`signature`) | Custom signature-gated IPC |

A custom permission `com.securitycom.permission.ACCESS` is declared at
`protectionLevel="signature"` — only apps signed with the same (OEM) key
may hold it. `android:usesCleartextTraffic="true"` and
`android:usesNonSdkApi="true"` are both set.

## Components (from manifest)

**Application:** `com.scorpio.ScorpioApplication`

**Activities**
- `com.scorpio.DownLoadActivity` — download UI (translucent), `singleTask`
- `com.scorpio.GuideActivity` — home entry (`com.securitycom.action.HOME`)
- `com.scorpio.BootCheckActivity` — `singleInstance`, 1-pixel style
  (`OnePxActivityStyle`); a known trick to keep the process alive.

**Services**
- `com.scorpio.service.PullService` (**exported**) — polls backend ("pull").
- `com.scorpio.service.SecurityComApiService` (**exported**,
  `com.securitycom.action.SERVICE_API`) — IPC API surface; contains
  `a(android.content.pm.Signature, String): String` → **caller-signature
  verification** (gates the API to trusted/system-signed callers).
- `com.scorpio.service.KeepAliveService` — persistent, guarded by
  `com.scorpio.securitycom.permission.ACCESS`.

**Receivers**
- `com.scorpio.control.AdminReceiver` — **Device Admin**
  (`BIND_DEVICE_ADMIN`, `@xml/device_admin`).
- `com.scorpio.receive.BootReceiver` — `BOOT_COMPLETED` +
  `transsion.action.START_BOOT`.
- `com.scorpio.receive.InstallAppReceiver` — install-session callbacks
  (`SESSION_API_PACKAGE_INSTALLED`).
- `com.scorpio.receive.MyPackageReplacedReceiver` — `MY_PACKAGE_REPLACED`.
- `com.scorpio.receive.MySecretCodeReceiver` — dialer `SECRET_CODE`
  trigger (hidden activation via a `*#*#code#*#*` dial code).
- `com.scorpio.control.FocusChangeReceiver`.

**Providers**
- `androidx.core.content.FileProvider` (`com.scorpio.fileprovider`) — serve
  staged APKs to the package installer.
- `com.liulishuo.okdownload.OkDownloadProvider` — the OkDownload library.

## Data model (the `com.scorpio.bean.*` classes)

These minified-but-named model classes map out the backend protocol:

| Bean | Inferred role |
|------|----------------|
| `RequestApkUrl` | Request an APK's download URL from the server |
| `VersionControlBean` | Version / update gating |
| `QueryAesKeyBean` | Negotiate an AES key for request/response crypto |
| `TeeEncryptData` | TEE (Trusted Execution Environment) encrypted payload |
| `TeeDeviceStatusInfo` | TEE-backed device attestation/status |
| `SdkAuthorizeBean` | SDK authorization handshake |
| `TrackBean` | Analytics / event tracking |
| `BaseBean`, `BaseBeanNew` | Generic API response envelopes |

## Libraries observed

- **okhttp3** (REST client; `META-INF/proguard/okhttp3.pro` present)
- **liulishuo OkDownload** (`com.liulishuo.okdownload.*`) — resumable downloads
- **androidx** (appcompat, core, lifecycle, fragment, …)

## Inferred behavior (NOT runtime-confirmed)

1. On boot / secret-code / IPC call, the app starts and keeps itself alive
   (`BootReceiver`, `KeepAliveService`, `BootCheckActivity`).
2. `PullService` contacts a backend; requests are protected with an AES key
   (`QueryAesKeyBean`) and TEE attestation (`Tee*` beans).
3. The server returns APK URLs / version info (`RequestApkUrl`,
   `VersionControlBean`).
4. OkDownload fetches the APK to external storage; `FileProvider` + 
   `REQUEST_INSTALL_PACKAGES` drive installation; `InstallAppReceiver`
   tracks the session result.
5. `SecurityComApiService` exposes a signature-gated IPC API to other
   (OEM-signed) apps. `AdminReceiver` gives the app device-admin powers.

## Security / privacy observations

- Capability to **silently fetch and install arbitrary APKs** from a
  server is the central risk; backend compromise or a malicious server
  response would turn into on-device app installation.
- **Device-admin** + **boot persistence** + 1px keep-alive make the
  component hard to remove and persistent across reboots.
- **Cleartext traffic allowed**; dialer **secret-code** hidden entry point.
- Exported `PullService` / `SecurityComApiService` are IPC attack surface,
  partially mitigated by signature checks.

## Reproduction (how this analysis was produced)

```bash
. ~/.local/rea-toolchain/env.sh            # JDK17 + apktool + jadx env
APK="apk/com.scorpio.securitycom_1.0.0.24-32_minAPI28(nodpi)_apkmirror.com.apk"
rea decode-android-resources "$APK"        # manifest + metadata
rea search-android-classes  "$APK" com.scorpio
rea inspect-android-class   "$APK" com.scorpio.service.SecurityComApiService
rea inspect-android-method  "$APK" <class> <method>   # per-method decompile
```

## Open items / next steps

- Decompile key methods (`inspect-android-method`) to recover backend
  hostnames/endpoints, the AES/TEE scheme, and the exact install flow.
- Map the exported `SERVICE_API` protocol (actions + extras).
- Decide publication scope & visibility (see project notes).
