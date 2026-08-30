# Typing Translate

Type in your language, press **space twice**, and the text is replaced in place
by its translation — in any Windows text field. Browsers, YouTube search,
Discord, Electron apps, terminals.

Translation runs **fully offline** on your machine. Nothing is ever sent to a
server.

```
selamat pagi semuanya␣␣   →   Good morning to you all.
```

---

## Download

**[Download the latest release →](../../releases/latest)**

1. Download `TypingTranslate-Setup-v1.0.0.exe`
2. Run it — choose where to install, and whether you want a desktop shortcut
3. Launch from the desktop or Start Menu
4. On first launch it downloads the translation model (~620MB, about a minute).
   Progress is shown in the tray tooltip and settings window.

No Python and no admin rights required — it installs per-user by default.

> **Windows SmartScreen will warn you.** The installer is unsigned
> (code-signing certificates cost several hundred dollars a year). Click
> *More info* → *Run anyway*. If that makes you uncomfortable — reasonable,
> given this app reads your keystrokes — build it yourself from source in two
> minutes; see [Build from source](#build-from-source).

**Requirements:** Windows 10/11 64-bit. An NVIDIA GPU is optional but makes it
~4x faster.

To uninstall: *Settings → Apps → Typing Translate*, or the Start Menu
uninstaller. You'll be asked whether to keep the downloaded model, so
reinstalling later doesn't re-download 620MB.

---

## Using it

Type normally, then tap **space twice** to translate what you just typed.
Also works after pasting text.

| | |
|---|---|
| **Translate** | double-space, or `Ctrl+Alt+T` |
| **Settings** | double-click the tray icon |
| **Pause** | right-click tray → *Enabled* |
| **Quit** | right-click tray → *Exit* |

The tray icon is a circular badge that shows state at a glance:

| Icon | Meaning |
|------|---------|
| 🟢 green | ready |
| 🟠 amber | loading / downloading model |
| 🔵 blue | translating |
| ⚪ grey | disabled |
| 🔴 red | error |

> **Can't find the tray icon?** Windows 11 hides new tray icons — click the
> **^** chevron next to the clock. Drag the icon onto the taskbar to pin it.

Pick your target language in **Settings**. 24 languages ship in the picker;
the underlying model supports 200.

---

## Privacy

This app installs a global keyboard hook, which is how it works in every
application. That deserves a plain explanation:

- **Nothing leaves your computer.** Translation runs locally. The only network
  request ever made is the one-time model download from HuggingFace.
- **Nothing is written to disk.** The typed-text buffer lives in memory and is
  cleared after every translation.
- **Password fields are skipped**, detected via the Windows accessibility API
  (`UIA_IsPasswordPropertyId`) on every focus change. Password managers
  (KeePass, 1Password, Bitwarden) are blocked by process name.
- **The buffer resets constantly** — on focus change, mouse click, arrow keys,
  Escape, Enter, and any Ctrl/Alt chord — so it never accumulates a transcript.
- **The source is here.** Read [`app/hooks.py`](app/hooks.py) and
  [`app/engine.py`](app/engine.py) and verify all of the above.

Your antivirus may still flag it: a global hook plus synthetic input plus
clipboard access is structurally what a keylogger looks like. That is an honest
description of the mechanism — the difference is what the program does with it.

---

## How it works

```
WH_KEYBOARD_LL hook (dedicated thread, returns in microseconds)
    │  enqueue only — never blocks
    ▼
shadow buffer  ── tracks what you typed, per focus context
    │  double-space (or Ctrl+Alt+T)
    ▼
NLLB-200-600M via CTranslate2      ~85ms GPU / ~300ms CPU
    │
    ▼
Shift+Left select old text  →  clipboard paste  →  restore clipboard
```

### Why a shadow buffer instead of reading the text box

The obvious design — ask UI Automation for the focused field's text — fails on
the apps that matter most:

| Framework | UIA read | UIA write |
|---|---|---|
| Win32 / WinForms / WPF | ✅ | ✅ |
| **Electron** (Discord, Slack) | ⚠️ limited | ⚠️ varies |
| **Console** (terminals) | ❌ | ❌ |
| WinUI 3 / WPF rich edit | ✅ | ❌ *by design* |

We never need to ask: the keystrokes were captured, so the text is already
known. UIA is used for exactly one thing it *is* reliable at — detecting
password fields.

---

## Performance

Measured on an RTX 3060 Ti, NLLB-600M int8:

| chars | CPU | GPU | speedup |
|------:|----:|----:|--------:|
| 4 | 318ms | 86ms | 3.70x |
| 15 | 243ms | 56ms | 4.32x |
| 53 | 386ms | 99ms | 3.89x |
| 182 | 1352ms | 335ms | 4.04x |

The GPU wins at **every** length — there is no crossover, so it is used for
everything when available. CPU-only machines work fine, just slower.

Reproduce: `python -m scripts.bench_device`

---

## Protected names

Translation models have no concept of a brand name, so multi-word product names
get reordered by the target grammar:

```
buka Hermes Agent sekarang   →   "Open up Agent Hermes now."     wrong
```

Protected terms are masked before translation and restored after:

```
buka Hermes Agent sekarang   →   "Open the Hermes Agent now."    correct
```

~100 app names ship by default. `@handles`, emails, URLs, filenames
(`config.json`) and version numbers (`2.4.0`) are always protected
automatically. Add your own in **Settings → Protected names**.

---

## Build from source

```bash
git clone https://github.com/YOUR_USERNAME/typing-translate.git
cd typing-translate

python -m venv .venv
.venv\Scripts\python.exe -m pip install -r requirements.txt
.venv\Scripts\python.exe app\main.py            # run directly

.venv\Scripts\python.exe scripts\build_release.py   # build the .zip
```

`--cpu-only` produces an archive ~770MB smaller without the CUDA runtime.

### Tests

```bash
.venv\Scripts\python.exe scripts\test_engine.py         # 29 - buffer, triggers, profiles
.venv\Scripts\python.exe scripts\test_glossary_unit.py  # 19 - name protection
.venv\Scripts\python.exe scripts\test_tray.py           #  7 - tray menu
.venv\Scripts\python.exe scripts\test_translate.py      # translation quality
.venv\Scripts\python.exe scripts\test_firstrun.py       # fresh-install path
```

`TT_DEBUG=1` enables engine tracing.

---

## Known limitations

- **Windows only.** The hook and injection layers are Win32-specific.
- **Cannot type into elevated windows** from a non-elevated process (Windows
  UIPI). Run as administrator if you need that.
- **Anti-cheat protected games** block synthetic input.
- **Multi-line text is not translated** — the buffer resets on Enter, because a
  linear erase cannot safely span lines.
- Auto language detection is a lightweight heuristic. For consistent results set
  the source language explicitly instead of leaving it on *Auto-detect*.

---

## Licence

Source code: **MIT** — see [LICENSE](LICENSE).

The bundled translation model (NLLB-200-distilled-600M) is **CC-BY-NC 4.0:
non-commercial use only**. The MIT licence covers this application's code, not
the model. For commercial use, swap in a permissively-licensed model such as
Opus-MT.
