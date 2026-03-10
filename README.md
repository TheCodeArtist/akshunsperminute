# AkshunsPerMinute

A lightweight Windows utility that counts your **akshuns per minute (APM)** — every keyboard keypress, mouse click, and scroll wheel tick — and displays the live number so you always know how fast you're moving.

---

## What is an "Akshun"?

| Input type | Counts as |
|---|---|
| Any keyboard keypress | 1 akshun |
| Left / right / middle mouse button click | 1 akshun |
| Mouse scroll wheel tick | 1 akshun |

Mouse movement alone does **not** count.

---

## How APM is Calculated

APM is computed over a **rolling 60-second window**: at any point in time the displayed number equals the total akshuns that occurred in the past 60 seconds, updated continuously.

---

## Display Modes

The tool supports two display modes — switchable at runtime:

| Mode | Description |
|---|---|
| **System Tray** | Sits quietly in the Windows system tray; the current APM value is shown as the tray icon label / tooltip. |
| **Overlay** | A small always-on-top floating window that renders the APM number over whatever you are doing. Transparency and position are configurable. |

---

## Requirements

- Windows 10 / 11
- Python 3.10+ *(only needed if running from source)*
- Dependencies (installed via `pip`):
  - [`pynput`](https://pypi.org/project/pynput/) — global keyboard & mouse listener
  - [`pystray`](https://pypi.org/project/pystray/) — system tray icon
  - [`Pillow`](https://pypi.org/project/Pillow/) — tray icon image rendering
  - [`tkinter`](https://docs.python.org/3/library/tkinter.html) — overlay window *(bundled with Python)*

---

## Running from Source

```bash
# Clone the repository
git clone https://github.com/your-username/AkshunsPerMinute.git
cd AkshunsPerMinute

# Install dependencies
pip install -r requirements.txt

# Run
python main.py
```

---

## Building a Standalone `.exe`

```bash
pip install pyinstaller
pyinstaller --onefile --windowed --name AkshunsPerMinute main.py
```

The compiled binary will appear in `dist/AkshunsPerMinute.exe`. No Python installation is required on the target machine.

---

## Project Structure

```
AkshunsPerMinute/
├── main.py            # Entry point
├── tracker.py         # Input listener & rolling APM calculation
├── tray.py            # System tray display mode
├── overlay.py         # Always-on-top overlay display mode
├── requirements.txt   # Python dependencies
└── README.md
```

---

## License

[Apache 2.0](LICENSE)
