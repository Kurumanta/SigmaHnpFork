# AGENTS.md — SigmaHnpFork workspace

## What this repo is

**Sigma** — a Vietnamese input method (bộ gõ tiếng Việt) for Windows. This repo is a **fork**
(`origin` = `Kurumanta/SigmaHnpFork`, `upstream` = `phamhoangnhat/Sigma`) and contains only the
**thin Qt shell app** (`Sigma.exe`): tray/config UI entry, single-instance mutex, admin elevation,
auto-update, and dynamic loading of the engine DLL.

**The actual input engine is NOT in this repo.** It lives in the sibling repo
`D:\02_Github_Repository\SigmaLibHNP` (fork of `phamhoangnhat/SigmaLib`), which builds
`SigmaLib.dll`. Any change to typing behavior, key hooks, or word composition happens there —
the `SigmaLib/` folder in this repo is **empty**; don't look for engine code here.
The two repos must be worked on together; place a freshly built `SigmaLib.dll` next to `Sigma.exe`.

## Architecture

Shell app (this repo):
- `main.cpp` — entry point. Single-instance via named mutex `"Sigma"`, ensures admin elevation,
  runs auto-update, then `LoadLibraryW("SigmaLib.dll")` and calls exports `sigmalib_start()` /
  `sigmalib_stop()`. If the DLL or those exports are missing it shows an error box and exits.
- `Update.cpp/h` — auto-update. Downloads `update.dat` from the **upstream** repo
  (`raw.githubusercontent.com/phamhoangnhat/Sigma/main/update.dat`); controlled by the
  `modeAutoUpdate` setting. A local fork build can be overwritten by upstream binaries if this is on.
- `AdminPrivilegeManager` — app always runs elevated (needed for low-level hooks + scheduled task).

Engine (`D:\02_Github_Repository\SigmaLibHNP`):
- `SigmaLib.cpp` — DLL exports `sigmalib_start` / `sigmalib_stop`.
- `Listener.cpp` — global **`WH_KEYBOARD_LL` + `WH_MOUSE_LL`** hooks + message pump; intercepts
  keystrokes before the focused app sees them.
- `Word.cpp` — Vietnamese orthography/composition (tone marks, vowels, spelling rules).
- `TypeWord.cpp` — **text rewrite engine**: composes the word and "fixes" already-typed text by
  synthesizing `SendInput` batches (VK_BACK / VK_LEFT / VK_RIGHT / Unicode chars).
- `CustomInputMethod.cpp` + `Variable.cpp` (`mapInputMethodBase`) — input methods **Telex** and **Vni**.
- `Variable.cpp/h` — global settings + per-app classification lists:
  - `listAppFixAutoSuggest` — apps where autosuggest-compat mode defaults ON (browsers, Office…).
    **`"notion"` (desktop) is NOT in this list** — but `notion.so` in Chrome is matched as
    `"chrome"`.
  - `listAppLangVietGlobal`, `listAppUseClipboard`.
- `ConfigUi.cpp` — per-app config dialog; the checkbox
  **"Tương thích với ứng dụng có gợi ý từ"** maps to setting `modeFixAutoSuggest`
  (per-account, per-app). Toggling has a global shortcut too.
- `AccountManager`, `TaskAI*`, `Snippet*`, `DictionaryViet` — accounts, AI tasks, snippets, dictionary.

## Settings storage

Qt `QSettings` → Windows registry `HKEY_CURRENT_USER\Software\Sigma`.
Most typing settings are per-account: group `AccountManager::currentAccount + "/Config"`;
per-app overrides live under `.../ConfigUi/<appName>` (app name = lowercase exe base name).
Global shell settings: `QSettings("Sigma", "Config")` (e.g. `modeAutoUpdate`, `verSigmaExe`).

## Build

- Visual Studio 2022 (v143 toolset), **x64 Debug / x64 Release only**, Qt **6.9.0**, Windows 10+.
- Both projects are `.sln`/`.vcxproj` based — open in Visual Studio (no CLI build scripts, no CMake
  as the primary path). Build SigmaLibHNP → copy `SigmaLib.dll` beside `Sigma.exe`.
- Version numbers are independent per repo: app `Sigma.rc` (recent commits: "Cập nhật phiên bản
  2.2.x"), engine `version.h` (`APP_VERSION_MAJOR/MINOR/PATCH`).

## Fixed issue: Vietnamese typing broke in Notion web code blocks (resolved 2026-10-14)

Root cause (proven by a marker-character diagnostic): Notion's code-block editor (and async web
editors in general) silently DROPS synthetic `VK_BACK` deletions when a whole `SendInput` batch
arrives within one frame, while `KEYEVENTF_UNICODE` inserts land fine. Sigma's single-batch
rewrite (`TypeWord::fixStringDisplay`) therefore under-deleted; the U+00B7 `·` bait from
`modeFixAutoSuggest` also survived, producing `d·đang` (checkbox on) / `dđang` (checkbox off).
Note: U+0008-as-unicode backspace is inert in Chrome — do not retry that approach.

Fix (SigmaLib 2.3.5): new per-app setting **`modeFixModernEditor`** ("Tương thích với trình soạn
thảo hiện đại", default ON for browsers via `listAppFixModernEditor` in `Variable.cpp`). When on,
`TypeWord::fixStringDisplay` sends deletions spread out — one `SendInput` per key with ~20 ms
sleeps (10 ms for very long rewrites) — instead of one batch. Everything else reverted to
upstream behavior. Wire-up: `Variable.{h,cpp}` (list + `loadSettingsWindow`), `ConfigUi.{h,cpp}`
(checkbox), `TypeWord.cpp`. Local builds use `CMakeLists.txt` + `build-sigma.ps1` (VS2022
BuildTools + Qt 6.9.0 via aqt are installed on this machine; Qt VS Tools are NOT).

## Conventions & gotchas

- Code comments, log messages, and UI strings are in **Vietnamese**; keep that style.
- Logging is `qDebug()`-based, messages in Vietnamese.
- Release binaries (`Sigma.exe`, `SigmaLib.dll`, `SigmaSetup.exe`, `Auto Sigma.rar`) are committed
  at repo root — don't treat their presence as build output to regenerate.
- Docs for end users: `Sigma User Guide.html` / `Sigma User Guide - EN.html` (also embedded in the
  engine repo). Update user-facing docs when changing behavior that the guide describes.
- App is single-instance and requires admin; for live testing, disable `modeAutoUpdate` first or a
  launch may fetch upstream binaries over your build.
