# Brawl Offline — клиент 31.96 (arm64-v8a)

Оффлайн-клиент Brawl Stars 31.96 (versionCode 142) с Frida-агентом, который
поднимает сервер прямо внутри процесса игры: логин, автосоздание профиля и
вход в главное меню без сети.

## Что внутри

- `out/BrawlOffline-v31-arm64.apk` — готовый arm64-v8a APK (181 МиБ),
  собран из официального `com.supercell.brawlstars 31.96 (arm64-v8a)`.
- `agent/core.js` — протокол, профиль, кодирование пакетов (побайтово
  совпадает с Python-сервером v31 из `src-v31/`).
- `agent/gadget.js` — клей Frida: перехват DNS (`getaddrinfo`,
  `android_getaddrinfofornet`) и `connect`, локальный TCP-сервер
  `127.0.0.1:9339`, мост `console.log` → logcat (тег `BrawlOffline`).
- `tools/build_apk.py` — сборка APK:
  - режим A — замена гаджета в готовых приватных сборках;
  - режим B — чистая инъекция: LIEF добавляет `DT_NEEDED
    libfrida-gadget.so` в `libg.so`, в `lib/<abi>/` кладутся
    `libfrida-gadget.so`, `libfrida-gadget.config.so` и
    `libagent.script.so`.
- `tools/bundle.py` — склейка `agent/*.js` и ассетов в
  `out/agent.bundle.js`.
- `tools/dev.sh` — hot-reload: пушит свежий бандл в
  `/sdcard/Android/data/<pkg>/files/brawl-offline/agent.js`, гаджет
  перечитывает его при изменении.
- `tools/run_waydroid.sh` — установка, запуск и logcat на Waydroid.
- `tests/harness.js` — эталонные пакеты и тесты протокола
  (`node tests/harness.js`).

## Сборка

```bash
# 1. Бандл агента
python3 tools/bundle.py

# 2. APK из официального arm64 клиента (нужен LIEF)
/tmp/opencode/venv/bin/python tools/build_apk.py \
  --in  /tmp/opencode/official-31.96-arm64.apk \
  --out out/BrawlOffline-v31-arm64.apk
```

Скрипт сам определяет ABI, добавляет `DT_NEEDED`, кладёт конфиг и скрипт,
выравнивает и подписывает debug-ключом (или `--ks`).

## Установка и запуск

```bash
adb install -r -d out/BrawlOffline-v31-arm64.apk
adb shell am start -n com.supercell.brawlstars/.GameApp
adb logcat -s BrawlOffline:V Frida:V
```

Или одной командой:

```bash
tools/run_waydroid.sh out/BrawlOffline-v31-arm64.apk com.supercell.brawlstars 192.168.240.112:5555
```

Логи агента также пишутся в
`/sdcard/Android/data/com.supercell.brawlstars/files/brawl-offline/agent.log`,
профиль — в `profile.json` рядом.

## Hot reload

```bash
tools/dev.sh com.supercell.brawlstars        # следит за agent/*.js и пушит изменения
```

Конфиг гаджета:

```json
{
  "interaction": { "type": "script", "path": "libagent.script.so", "on_change": "reload" },
  "runtime": "v8",
  "log": { "level": "info", "handler": "native" }
}
```

## Важно про эмуляторы

- **Waydroid (x86_64 + houdini/ndk-translation)**: arm64-код транслируется,
  поэтому холодный старт официального клиента на Java-инициализации
  (Sentry/Firebase) занимает минуты и система показывает ANR
  «Brawl Stars isn't responding». Это не ошибка сборки — на реальном
  arm64-устройстве этой задержки нет. Если ANR появился, нажмите «Wait»:
  процесс продолжает работу и в итоге доходит до `libg.so`.
- Для проверки логина/меню нужен реальный arm64-телефон (или эмулятор с
  arm64-системой).
