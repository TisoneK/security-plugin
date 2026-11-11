# Environments (update in place)

Machines and sandboxes agents have run on, and what it takes to work on
this project from each. One block per environment; update the matching
block (and its "last verified" date) every time you run on it again.

## Rules

1. **Match before you add.** At session start, check whether the machine
   you're on already has a block (use its "Identify by" line). Update the
   match; add a new block only for a genuinely new environment.
2. **Record what you verified, not what you assume.** A command belongs
   under "Verified commands" only after it ran successfully on this
   environment, this project.
3. **Agents never delete blocks.** An environment the project no longer
   uses may be pruned by the user; if you can't verify a block, leave it
   alone — its last-verified date already says how stale it is.
4. **Machine facts only.** Secret values go in `secrets/`; user
   preferences in `user/`; project-wide decisions in `plans/`.

<!-- TEMPLATE — one block per environment:
---
## <stable label — hostname, "Z sandbox", "GitHub Actions ubuntu-24.04"> (last verified YYYY-MM-DD)
- **Identify by:** <how an agent recognizes this env — hostname, $USER, workspace path>
- **OS:** <e.g., macOS 15.5 / Ubuntu 24.04 sandbox>
- **Runtimes:** <node X, python Y, ...>
- **Package manager:** <npm/bun/pnpm/pip/...>
- **Verified commands:** <install / test / lint / typecheck / dev-server commands that actually worked here, with cwd if it matters>
- **Quirks:** <e.g., "no psql installed", "port 3000 usually taken", "system Python locked down">
-->

---
## Tisone's Mac — Intel (last verified 2026-11-11)
- **Identify by:** `uname -m` = `x86_64` AND `sysctl -n machdep.cpu.brand_string` = `Intel(R) Core(TM) i3-8100B`; user `bao`; project at `/Users/bao/Code/security-plugin`.
- **OS:** macOS (Darwin 24.6.0), Intel x86_64.
- **Runtimes:** node 24.22.2. **No system Java** — Android Studio ships an **arm64** JBR that will NOT run on this Intel host. No Homebrew, no androguard.
- **REA toolchain (required for Android analysis):** built at `~/.local/rea-toolchain/` — Temurin **JDK 17 x64**, `apktool_3.0.3.jar` + `bin/apktool` wrapper, `jadx-headless-mcp-0.7.1-all.jar`. **Always `source ~/.local/rea-toolchain/env.sh` before any `rea` command** (sets JAVA_HOME, PATH, REA_APKTOOL_COMMAND, REA_JADX_MCP_JAR). This folder is OUTSIDE the repo and is not version-controlled.
- **Verified commands:** `source ~/.local/rea-toolchain/env.sh && rea doctor` → apktool + jadx healthy; `rea decode-android-resources <apk>`; `rea search-android-classes <apk> <query>`; `rea inspect-android-class <apk> <fqcn>`.
- **Quirks:** REA `decompile` (native providers) does NOT handle APK archives — use the `*-android-*` commands. Ghidra/IDA/Kuna providers not configured. gh CLI authenticated as `TisoneK`.

