# OFFLINE BRAWL STARS CLIENT — PROMPT FOR NEXT AI SESSION

## 🎯 Goal

Build an **offline Brawl Stars client** (game protocol emulation via Frida agent + local TCP server) for **Android arm64-v8a**, v31.96 (versionCode 142), and make it run on **Waydroid**.

Final artifact must be **arm64-v8a** (user rejected arm32). Agent must work both:
- embedded via Frida gadget in `libg.so` (mode B clean injection), and
- via `frida -U -f <pkg> -l agent.js` (external).

Requirements: hot reload of JS without APK rebuild, very detailed in-client logging, auto profile, login, main menu.

## 📌 Current state (as of 2026-11-01, session on Waydroid)

### ✅ Working
- Stock arm64 APK verified: `/tmp/opencode/official-31.96-arm64.apk` — 158506865 bytes, versionName 31.96, versionCode 142, native-code `arm64-v8a`, extractNativeLibs=true, targetSdk 29.
- Agent code: `agent/gadget.js` (Frida glue, DNS/connect hooks, TCP server 127.0.0.1:9339, logcat bridge tag `BrawlOffline`), `agent/core.js` (protocol/profile/encoders, byte-exact vs Python v31 server). Agent bundle: `out/agent.bundle.js` (~58.5 KiB, built via `tools/bundle.py`).
- **KEY FIX**: Frida gadget runtime switched `"v8"` → `"qjs"` (QuickJS). V8 JIT under houdini (Waydroid ndk-translation) burns a CPU core for minutes and never runs the script; QJS interpreter starts instantly. Decisive fix for Waydroid arm64.
- Rebuilt patched arm64 APK: `/home/w1ckedoneshot/brawl-offline/out/BrawlOffline-v31-arm64.apk` (mode B: `DT_NEEDED libfrida-gadget.so` added to `libg.so`, config `libfrida-gadget.config.so`, script `libagent.script.so`, runtime `"qjs"`). Built via `python3 tools/build_apk.py --in /tmp/opencode/official-31.96-arm64.apk --out out/BrawlOffline-v31-arm64.apk`.
- `tools/build_apk.py` fixed for LIEF 1.0.0-dev: `binary.dynamic_libraries` → `list(binary.libraries)`.
- Agent successfully booted on Waydroid arm64 under houdini (logcat tag `BrawlOffline`): `runtime:"QJS"`, `boot: ready`, `server: listening on 127.0.0.1:9339`, `net: connect hook installed`, `dns: getaddrinfo hook installed`.
- Agent bundle API compat with QJS verified (timers `setInterval`/`setTimeout` exist; `TextEncoder` not needed).

### ❌ Blocker (arm64 native under Waydroid)
- After agent boot, process dies ~2s later: `libhoudini.so` SIGSEGV, code 1 (SEGV_MAPERR), frames all in `/system/lib64/libhoudini.so`, `meet illegal address ... without executable permission`, `rax=0xdead1007`, backtrace BuildId `a8f9eb16786fb84cf49ef2a32078627220de72bb`. ANR fires on `com.supercell.brawlstars/com.google.android.gms.measurement.AppMeasurementService`.
- **Stock official arm64 APK crashes identically** → problem is houdini/ndk-translation arm64 incompatibility, NOT our gadget.

### 📱 Waydroid environment facts (test env)
- Device: Waydroid; adb serial `192.168.240.112:5555` (`adb -s 192.168.240.112:5555`, must `adb connect`).
- `waydroid status` 2026-11-01: Session RUNNING, Container RUNNING, Vendor MAINLINE, IP 192.168.240.112, user w1ckedoneshot(1000).
- Session: run `waydroid session start` as background task (never under `timeout`); after `adb reboot` container stops → must restart session manually.
- Props: `ro.hw_timeout_multiplier=10` (via `waydroid prop set`, effective after Android reboot), `ro.dalvik.vm.native.bridge=libhoudini.so`, `ro.enable.native.bridge.exec64=1`, `ro.vendor.enable.native.bridge.exec=1`, `ro.vendor.enable.native.bridge.exec64=1`, `ro.ndk_translation.version=0.2.3`, `ro.dalvik.vm.isa.arm64=x86_64`, `ro.product.cpu.abilist64=x86_64,arm64-v8a`.
- houdini version: `14.0.0_z.GoogleGame_com1.2`; waydroid version 1.6.3; vendor MAINLINE; abilist `x86_64,x86,arm64-v8a,armeabi-v7a,armeabi`.
- `settings put global hide_error_dialogs 1` was BAD → AMS auto-kills app on ANR (`am_kill ... user request after error`). Must keep `hide_error_dialogs=0`.
- Raise service timeout: `settings put global activity_manager_constants service_timeout=1200000,service_background_timeout=1200000,...`.
- Tools: `tools/waydroid_tune.sh`, `tools/waydroid_watch.sh` (dismiss ANR via `input tap 470 559` "Wait", wait for agent logs).

### 📦 armv7 findings
- Patched armv7: `out/BrawlOffline-v31-armv7.apk` (mode A, replaces `libmg.so` in private client `com.cb.brawlstars.v31`, runtime v8) — also stuck at Firebase `AppMeasurementService` init before reaching libg.
- Private armv7 client `/tmp/opencode/cb-v31.apk` (`com.cb.brawlstars.v31`) runs fine on this Waydroid.

### 🔄 User context
- User updated Waydroid recently — re-check `/system/lib64` libs and props after update.
- **Confirmed decision**: `/home/w1ckedoneshot/Загрузки/Telegram Desktop/BlockBuster v0.2(SMP).apk` (169391851 bytes, v0.2 SMP, v30-based, arm7+arm8) will be used as the **BASE** for the v31 adaptation — needs adaptation from v30 to v31. Copy of this context saved at `/home/w1ckedoneshot/Загрузки/Telegram Desktop/BrawlOffline_AGENT_PROMPT.md`.

## 🔜 Candidate next steps
1. After Waydroid update: check `adb -s 192.168.240.112:5555 shell ls -la /system/lib64/libhoudini.so /system/lib64/libndk_translation.so`, getprop values; if new `libndk_translation.so` exists → retest arm64.
2. If arm64 still broken → go armv7 route: strip Firebase/GMS (`AppMeasurementService`, `FirebaseInitProvider`, measurement) from manifest/dex.
3. Else investigate houdini arm64 crash fix (OTA image, props).

## 📚 Source links
- Frida docs: https://frida.re/docs/home/
- Frida gadget: https://frida.re/docs/gadget/
- Frida gadget environment (runtime): https://frida.re/docs/gadget/environment/
- Frida GitHub: https://github.com/frida/frida
- Waydroid docs: https://docs.waydro.id/
- Waydroid GitHub (images, houdini): https://github.com/waydroid/waydroid
- Android NDK runtime transport: https://developer.android.com/training/articles/ndk-transport

## 📱 APK v31 locations
- Stock v31.96 arm64: `/tmp/opencode/official-31.96-arm64.apk` (158506865 B)
- Patched v31 arm64 (gadget, qjs): `/home/w1ckedoneshot/brawl-offline/out/BrawlOffline-v31-arm64.apk` (190111779 B)
- Patched v31 armv7: `/home/w1ckedoneshot/brawl-offline/out/BrawlOffline-v31-armv7.apk`
- Private v31 armv7 client: `/tmp/opencode/cb-v31.apk`
- BlockBuster v0.2(SMP) alt base: `/home/w1ckedoneshot/Загрузки/Telegram Desktop/BlockBuster v0.2(SMP).apk`

---

## 🧾 Prompt to paste into a new AI session

```
We are building an OFFLINE BRAWL STARS CLIENT (v31.96, versionCode 142, arm64-v8a) with an in-process Frida agent + local TCP game server, running on Android via Waydroid.

Hard requirements:
- Final APK must be arm64-v8a (no arm32).
- Agent must work embedded (Frida gadget in libg.so, mode B) AND via `frida -U -f <pkg> -l agent.js`.
- Hot reload of JS without APK rebuild; very detailed in-client logging; auto profile; login flow; main menu.

Current verified state:
- Stock APK: /tmp/opencode/official-31.96-arm64.apk (arm64-v8a, versionCode 142, verified).
- Patched arm64 build: /home/w1ckedoneshot/brawl-offline/out/BrawlOffline-v31-arm64.apk — gadget runtime switched to "qjs" (QuickJS); agent BOOTS on Waydroid and starts TCP server on 127.0.0.1:9339 (logcat tag "BrawlOffline", runtime "QJS", boot: ready).
- BLOCKER: arm64 process still dies ~2s after agent boot inside /system/lib64/libhoudini.so (SIGSEGV SEGV_MAPERR, rax=0xdead1007, "meet illegal address ... without executable permission"). Stock official arm64 APK crashes identically → houdini/ndk-translation arm64 issue, not our gadget.

Environment:
- Waydroid test env, adb -s 192.168.240.112:5555 (adb connect 192.168.240.112:5555).
- waydroid 1.6.3, vendor MAINLINE, houdini 14.0.0_z.GoogleGame_com1.2, ro.ndk_translation.version=0.2.3, ro.hw_timeout_multiplier=10, ro.enable.native.bridge.exec64=1.
- Must NOT set global hide_error_dialogs=1 (AMS auto-kill on ANR). Keep 0.
- Session: run `waydroid session start` as background task; after `adb reboot` container stops → restart session.

Open questions / tasks:
1. Re-check /system/lib64 after user's Waydroid update for libndk_translation.so — retest arm64.
2. If arm64 still crashes → switch to armv7 path: strip Firebase/GMS components (AppMeasurementService / FirebaseInitProvider) from manifest + dex, using out/BrawlOffline-v31-armv7.apk (mode A, private client com.cb.brawlstars.v31 base from /tmp/opencode/cb-v31.apk) as base.
3. Alternative: /home/w1ckedoneshot/Загрузки/Telegram Desktop/BlockBuster v0.2(SMP).apk (v30-based, arm7+arm8) adapted to v31.

Key project files:
- /home/w1ckedoneshot/brawl-offline/agent/gadget.js — agent logic (hooks, server, logging)
- /home/w1ckedoneshot/brawl-offline/agent/core.js — protocol/profile/encoders
- /home/w1ckedoneshot/brawl-offline/tools/build_apk.py — mode A/B build, LIEF fix, runtime qjs
- /home/w1ckedoneshot/brawl-offline/tools/bundle.py — agent.bundle.js builder
- /home/w1ckedoneshot/brawl-offline/tools/waydroid_tune.sh, tools/waydroid_watch.sh — Waydroid AMS/ANR tuning + ANR "Wait" dismiss
- /home/w1ckedoneshot/brawl-offline/README.ru.md

Prior decisive findings:
- Frida runtime v8 → qjs was THE fix that got the agent executing under Waydroid houdini.
- arm64 native crash reproducible with stock APK → houdini-level blocker.

Continue: verify post-update Waydroid state, then decide arm64 fix vs armv7 no-GMS path. Report honestly at the end.
```

## 🔗 Source links (ready to paste)
- Frida docs: https://frida.re/docs/home/
- Frida gadget: https://frida.re/docs/gadget/
- Frida gadget runtime/env: https://frida.re/docs/gadget/environment/
- Frida GitHub: https://github.com/frida/frida
- Waydroid docs: https://docs.waydro.id/
- Waydroid GitHub: https://github.com/waydroid/waydroid
- Android NDK runtime transport: https://developer.android.com/training/articles/ndk-transport

## 📲 Brawl Stars v31 APKs
- **Stock v31.96 arm64 (official)**: `/tmp/opencode/official-31.96-arm64.apk` — 158,506,865 bytes
- **Patched v31 arm64 (our build, gadget=qjs)**: `/home/w1ckedoneshot/brawl-offline/out/BrawlOffline-v31-arm64.apk` — 190,111,779 bytes
- **Patched v31 armv7**: `/home/w1ckedoneshot/brawl-offline/out/BrawlOffline-v31-armv7.apk`
- **Private v31 armv7 client**: `/tmp/opencode/cb-v31.apk` (285,182,330 B)
- **Alt base BlockBuster v0.2(SMP)**: `/home/w1ckedoneshot/Загрузки/Telegram Desktop/BlockBuster v0.2(SMP).apk` (169,391,851 B)

---

**Итоговый файл промпта**: записан в `/home/w1ckedoneshot/brawl-offline/PROMPT.md` (это и есть «промпт в один файл» — содержит и контекст, и готовый текст для вставки в новую сессию).
