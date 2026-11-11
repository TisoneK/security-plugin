# Session 2026-11-11-1 notes

## What happened
- Copied the target APK into `apk/` (sha256 `35ec8526…a14fef`), git-init'd the repo, bootstrapped context-ledger core 2.3.3.
- **Environment blocker:** this is a genuine **Intel Mac**. The only Java present was Android Studio's **arm64** JBR, which cannot run on Intel ("Bad CPU type"). REA's Android pipeline (apktool + jadx) is JVM-based, so it was fully blocked.
- **Fix (user-level, no system install):** downloaded into `~/.local/rea-toolchain/`:
  - Temurin JDK 17.0.21 (macOS **x64**) — extracted, runs on Intel.
  - `apktool_3.0.3.jar` + a wrapper script `bin/apktool`.
  - `jadx-headless-mcp-0.7.1-all.jar` (sha256 verified against published hash).
  - `env.sh` exports `JAVA_HOME`, `PATH`, `REA_APKTOOL_COMMAND`, `REA_JADX_MCP_JAR`.
  - `rea doctor` then reports apktool + jadx providers **healthy**.
- Ran REA: `decode-android-resources` (manifest), `search-android-classes` (754 classes, 24 app-owned), `inspect-android-class` on core services. Wrote `analysis/ANALYSIS.md`.

## Key finding
`com.scorpio.securitycom` is a **Transsion (Tecno/Infinix/itel) pre-installed OEM
"SecurityCom" component** that pulls APK URLs from a backend and **installs
APKs** — with device-admin, boot persistence, keep-alive, TEE-backed AES
server comms, a dialer secret-code entry point, and a signature-gated exported
IPC API. Code is ProGuard-minified (method names `a`,`b`…).

## Why we paused
"Recreate and publish to GitHub" (owner's request) targets **proprietary OEM
code**. A clean buildable clone isn't achievable (obfuscated) or advisable
(copyright + the app has silent-install/persistence capabilities). The
defensible deliverable is a **documented RE analysis**, and the repo should
likely be **private**. Needs owner decision — see tasks/parking-lot.md.

## To reproduce REA next session
`source ~/.local/rea-toolchain/env.sh` first, then `rea <cmd>`. Without it REA
cannot find Java/apktool/jadx. The toolchain lives OUTSIDE the repo (in `~`),
so a fresh clone on another machine must rebuild it.
