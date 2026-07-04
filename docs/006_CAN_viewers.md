# Roundup of the best open source tools for viewing DBC-decoded CAN traces (BLF, TRC, ASC, etc.):

---

### 1. **SavvyCAN** *(C++/Qt — cross-platform)*
Probably the most feature-rich free GUI tool for this purpose. It supports DBC signal decoding with a treeview, signal viewer, and temporal view, and lets you search for signals and messages by name or ID. It supports TRC (CanHacker format) and can load BLF files as well. Great for both offline trace analysis and live capture. Also has reverse-engineering tools (fuzzing, flow view, etc.).
- GitHub: `collin80/SavvyCAN`

---

### 2. **BUSMASTER** *(C++ — Windows)*
An open source tool to simulate, analyze, and test CAN data bus systems. It's a fairly close open-source analog to CANalyzer, with DBC support, signal decoding, logging, and replay. There's also a fork called **Novo Bus Analyzer** with some additional features. Windows-only, but mature and well-established.
- GitHub: `rbei-etas/busmaster`

---

### 3. **CANgaroo** *(C++/Qt — cross-platform)*
Supports loading multiple DBC files for CAN signal decoding, and has integrated graphing tools supporting time-series, scatter charts, text-based monitoring, and interactive gauge views. It also has a built-in Python scripting engine (via pybind11) for automation. Actively maintained fork with CAN FD support.
- GitHub: `Schildkroet/CANgaroo`

---

### 4. **CAN++** *(C++ — Windows)*
A free Windows program for receiving, transmitting, and analyzing CAN bus messages (CAN Classic and CAN FD). It supports importing DBC and ARXML databases, shows signals in symbolic form, and can display signals as waveforms. BLF files can be imported, and ASC traces can be generated and replayed.
- GitHub: `TDahlmann/canpp`

---

### 5. **asammdf GUI** *(Python — cross-platform)*
A 100% free open source tool for processing MF4/MDF data. It lets you load, analyze, DBC decode, and plot CAN/LIN data, create visual data plots including calculated fields, filter data in a tabular view, or export to CSV/MAT. Primarily MF4-focused, but can convert to/from TRC and other formats. Very good for signal plotting.
- GitHub: `danielhrisca/asammdf`

---

### 6. **cantools** *(Python — CLI + library)*
Supports DBC, KCD, SYM, ARXML file parsing and encoding/decoding of CAN frames, and can be used with python-can which supports various popular CAN bus file formats like Vector ASC, BLF, and TRC. Not a GUI, but extremely powerful as a scripting foundation — ideal if you want to build your own tooling or do batch processing.
- GitHub: `eerimoq/cantools` + `hardbyte/python-can`

---

### 7. **Canverter Pro** *(Python/PySide6 — cross-platform)*
A free, open-source, offline desktop app built on cantools and python-can, with a PySide6 + Matplotlib GUI. It reads `.asc`, `.blf`, `.trc`, `.log` files, decodes them using DBC files, and exports to CSV with optional resampling and graph visualization. Newer project (2026) but fills the gap between full GUI tools and raw Python scripts nicely.
- GitHub: https://github.com/birol91/canverter_desktop_app

---

### Quick format support summary

| Tool | BLF | TRC | ASC | DBC | GUI | Platform |
|---|---|---|---|---|---|---|
| SavvyCAN | ✓ | ✓ | ✓ | ✓ | ✓ | Win/Lin/Mac |
| BUSMASTER | ✓ | ✓ | ✓ | ✓ | ✓ | Windows |
| CANgaroo | partial | — | ✓ | ✓ | ✓ | Win/Lin/Mac |
| CAN++ | ✓ | — | ✓ | ✓ | ✓ | Windows |
| asammdf | — | ✓* | — | ✓ | ✓ | Win/Lin/Mac |
| cantools | ✓ | ✓ | ✓ | ✓ | CLI | Any |
| Canverter Pro | ✓ | ✓ | ✓ | ✓ | ✓ | Win/Lin/Mac |

For a full GUI on Linux with solid BLF+TRC+DBC support, **SavvyCAN** is likely your best starting point. For heavy signal plotting and analysis, **asammdf** pairs very well with it.