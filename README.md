# Çalgı Çengi (Ritim Ustası)

> A touchscreen rhythm sequencer that plays real percussion instruments through solenoids driven by an ESP32-S3.

> 🇹🇷 **Türkçe özet:** Çalgı Çengi, dokunmatik ekranda 9 slotluk bir ritim dizisi oluşturup bunu ESP32-S3 üzerinden gerçek davul, zil, darbuka ve ksilafonu vuran solenoidlerle çalan bir sergi/kiosk projesidir. Arayüz tarayıcıda (React + Web Serial API), donanım yazılımı ise PlatformIO ile yazılmış C++ firmware'dir.

## Overview

Visitors build a rhythm on a touch display by dragging instrument cards into nine slots and choosing how many times each instrument hits. The browser app then drives eight solenoids over USB, so the instruments are struck physically: two bass drums, two cymbals, a darbuka, a drum-kit drum, a mini drum and a xylophone. Preset rhythms are included, and custom rhythms can be saved on the device.

It was built as an exhibit: a Windows kiosk launcher starts the app full screen, and the UI blocks the shortcuts a visitor could use to leave it. The project is archived.

## Features

- Nine-slot rhythm builder with drag and drop (mouse and touch sensors tuned separately), per-slot hit counts and a "wait" block for rests
- Adjustable BPM, preset rhythms (march, rock, jazz, darbuka, xylophone, and others) and a melody browser
- Rhythms can be saved to and loaded from the ESP32's flash (LittleFS) over the serial link
- Paired instruments (bass drum, cymbal) alternate between two solenoids so each coil gets twice the recovery time
- PIN-protected settings panel for the hardware connection and BPM
- Browser-side audio preview tones when no hardware is connected
- Kiosk hardening: context menu and refresh/close shortcuts are blocked in the page, and Windows edge-swipe gestures can be disabled with `kiosk-lock.bat`

## Architecture

```text
 Touch UI (React, Vite)                       ESP32-S3 firmware (PlatformIO)
┌────────────────────────┐   USB CDC serial   ┌──────────────────────────────┐
│ App.tsx  slots, timing │  115200 baud       │ serial_parser  line commands │
│ serialBridge.ts        │ ─────────────────► │ solenoid       pulse timers  │
│  Web Serial API        │  HIT:ch[:ms]       │ sequencer      step engine   │
│                        │  STOP, SAVE:name   │ storage        LittleFS JSON │
│                        │ ◄───────────────── │                              │
└────────────────────────┘  READY / PONG / OK └──────────────────────────────┘
                                                      │ GPIO
                                                      ▼
                                      2 x ULN2803A drivers -> 8 solenoids
```

The browser owns the timing of the rhythm and sends `HIT` commands over a line-based ASCII protocol (`HIT`, `SEQ`, `STOP`, `SETPULSE`, `SAVE`, `LOAD`, `LIST`, `DELETE`, `PING`). The firmware replies with short status lines (`READY`, `PONG`, `OK`, `PLAYING`, `ERR:...`). The firmware also contains an on-device step sequencer (`SEQ`) for patterns of up to 64 steps.

## Tech stack

| Layer | Technology |
|---|---|
| UI | React 19, TypeScript, Vite 6, Tailwind CSS 4, dnd-kit, motion |
| Browser-to-device link | Web Serial API (Chrome / Edge) |
| Firmware | C++ on Arduino framework for ESP32-S3, PlatformIO, ArduinoJson, LittleFS |
| Hardware | ESP32-S3 DevKitC-1, 2 x ULN2803A Darlington drivers, 8 solenoids |
| Kiosk | Windows batch scripts, Microsoft Edge kiosk mode |

## Engineering highlights

- **Non-blocking firmware:** `loop()` only calls `SerialParser::update()`, `Sequencer::update()` and `Solenoid::update()`; each solenoid has its own end-time, so overlapping hits never block each other.
- **Coil protection:** pulse length is clamped between 5 and 300 ms in firmware, regardless of what the host sends.
- **Robust serial connection:** the bridge sets DTR/RTS manually (the S3's native USB CDC needs it), guards `setSignals()` with a timeout, re-sends `PING` after connecting in case the one-shot `READY` was missed, and cleans up half-opened ports so the next attempt does not hang.
- **Auto-reconnect on boot:** previously authorised ports are reopened automatically; the kiosk launcher pre-grants the Edge serial-port policy so no port picker appears.
- **Bit-packed patterns:** each channel's steps are stored as a 64-bit mask, and saved melodies are JSON files on LittleFS with validated names.
- **Kiosk launcher:** `start-kiosk.bat` builds the app if needed, serves it with `vite preview`, waits for the server, hides the Windows shell, opens Edge in kiosk mode and restores the shell when Edge exits.

## Project status

Completed / archived. The software and firmware target the physical instrument rig described above (pin map in `firmware/src/config.h`). There are no automated tests in the repository.

## Repository layout

```text
src/App.tsx               # Main UI: slots, drag and drop, playback timing
src/components/           # SettingsPanel (PIN + connection + BPM), MelodyBrowser
src/core/                 # serialBridge, instruments, melody model, preset rhythms
firmware/src/             # main, serial_parser, solenoid, sequencer, storage, config.h (pin map)
firmware/platformio.ini   # ESP32-S3 DevKitC-1 build configuration
public/instruments/       # Instrument card images
muzik_aleti_icon/         # Source instrument icons
start-kiosk.bat           # Windows kiosk launcher (Edge, full screen)
kiosk-lock.bat            # Disable touch edge gestures and window snapping (admin)
kiosk-unlock.bat          # Revert the lock
```

## Running locally

Requires Node.js, PlatformIO, and Chrome or Edge (for Web Serial). The UI works without hardware; the device is only needed for real playback.

```bash
npm install
pio run -d firmware --target upload   # flash the ESP32-S3
npm run dev                           # UI at http://127.0.0.1:3000
```

For the exhibit setup on Windows, run `kiosk-lock.bat` once (as administrator, then sign out and back in), then start `start-kiosk.bat`.
