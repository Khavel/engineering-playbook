# Python PyQt6 Game Overlay with OCR

Building a Windows game overlay that uses OCR to extract text from screenshots and display real-time information.

## Architecture Pattern

`
Hotkey Listener (keyboard lib)
       │
       ▼
Screen Capture (mss - thread-safe)
       │
       ▼
OCR Processing (Tesseract via pytesseract)
       │
       ▼
API Lookup (httpx async)
       │
       ▼
Qt Overlay (PyQt6 - main thread only)
`

## Key Learnings

### 1. Qt Threading is Critical

**Problem**: Qt GUI operations MUST happen on the main thread. Calling Qt from background threads causes crashes or silent failures.

**Solution**: Use a queue-based approach:

`python
import queue
from PyQt6.QtCore import QTimer

class App:
    def __init__(self):
        self._ui_queue = queue.Queue()
    
    def _queue_ui_update(self, action: str, *args):
        self._ui_queue.put((action, args))
    
    def _process_ui_queue(self):
        try:
            while True:
                action, args = self._ui_queue.get_nowait()
                if action == "show_results":
                    self.overlay.show_results(args[0])
        except queue.Empty:
            pass

# Set up timer in Qt event loop
timer = QTimer()
timer.timeout.connect(self._process_ui_queue)
timer.start(50)  # Process every 50ms
`

### 2. mss for Screen Capture

**Problem**: PIL ImageGrab is slow and not thread-safe.

**Solution**: Use mss library - create fresh instance per thread:

`python
import mss

def capture_screen(monitor_index: int = 0):
    with mss.mss() as sct:  # Fresh instance per call
        monitors = sct.monitors
        monitor = monitors[monitor_index + 1]  # 0 is "all monitors"
        screenshot = sct.grab(monitor)
        return Image.frombytes("RGB", screenshot.size, screenshot.bgra, "raw", "BGRX")
`

### 3. Multi-Monitor Detection

**Problem**: User might be gaming on any monitor.

**Solution**: Detect mouse position and capture that monitor:

`python
import mouse

def get_monitor_at_mouse():
    x, y = mouse.get_position()
    with mss.mss() as sct:
        for i, mon in enumerate(sct.monitors[1:], 1):
            if (mon["left"] <= x < mon["left"] + mon["width"] and
                mon["top"] <= y < mon["top"] + mon["height"]):
                return i
    return 1
`

### 4. OCR Spatial Detection

**Problem**: OCR returns all text, including UI elements. Need to identify player names specifically.

**Solution**: Use spatial relationships - find anchor labels (DOWNED, VOIP) and look for names ABOVE them:

`python
def extract_players(ocr_results):
    # Find interaction labels
    anchors = [r for r in ocr_results if r.text.lower() in {"downed", "voip"}]
    
    players = []
    for anchor in anchors:
        ax1, ay1, ax2, ay2 = anchor.bbox
        
        # Find text ABOVE this anchor (smaller Y value)
        for result in ocr_results:
            tx1, ty1, tx2, ty2 = result.bbox
            
            # Must be above (ty2 < ay1)
            if ty2 >= ay1:
                continue
            
            # Must be horizontally aligned
            x_overlap = max(0, min(tx2, ax2) - max(tx1, ax1))
            if x_overlap < 20:
                continue
            
            # Must be close (within 100px)
            if ay1 - ty2 > 100:
                continue
            
            players.append(result.text)
    
    return players
`

### 5. Filter Lists for OCR

**Problem**: OCR picks up game UI, Windows UI, browser text, even the overlay itself.

**Solution**: Maintain comprehensive filter lists:

`python
filter_words = {
    # Game UI
    "player", "interactions", "team", "squad", "voip", "downed",
    # Windows UI
    "edit", "file", "view", "help", "search", "close", "minimize",
    # Overlay's own text (prevent recursion!)
    "reputations", "neutral", "trusted", "create", "dismiss",
    # Resolution patterns
}

# Also filter by pattern
if re.match(r'^x\d{3,4}$', text):  # x1440, x1080
    continue
`

### 6. Global Hotkeys on Windows

**Problem**: pynput is unreliable on Windows, especially with games.

**Solution**: Use keyboard library:

`python
import keyboard

def setup_hotkeys():
    keyboard.add_hotkey("ctrl+shift+r", on_capture, suppress=True)
    keyboard.add_hotkey("ctrl+shift+q", on_quit, suppress=True)
`

### 7. Always-on-Top Transparent Overlay

`python
from PyQt6.QtCore import Qt
from PyQt6.QtWidgets import QWidget

class Overlay(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowFlags(
            Qt.WindowType.WindowStaysOnTopHint |
            Qt.WindowType.FramelessWindowHint |
            Qt.WindowType.Tool  # Don't show in taskbar
        )
        self.setAttribute(Qt.WidgetAttribute.WA_TranslucentBackground)
        self.setWindowOpacity(0.95)
`

## Dependencies

`	oml
[project]
dependencies = [
    "PyQt6>=6.6.0",
    "pytesseract>=0.3.10",
    "Pillow>=10.0.0",
    "mss>=9.0.0",
    "keyboard>=0.13.0",
    "mouse>=0.7.0",
    "httpx>=0.27.0",
    "pydantic>=2.0.0",
    "pydantic-settings>=2.0.0",
]
`

## Tesseract Installation (Windows)

`powershell
winget install UB-Mannheim.TesseractOCR

# Add to PATH if needed
[Environment]::SetEnvironmentVariable(
    "Path", 
    $env:Path + ";C:\Program Files\Tesseract-OCR", 
    [EnvironmentVariableTarget]::User
)
`

## Anti-Cheat Considerations

Screen capture is safe because it:
- Only reads pixels from display output (same as OBS, Discord)
- Does NOT read game memory
- Does NOT inject code
- Does NOT hook into game APIs

This is the same technology streaming software uses.

## Reference Implementation

See: RaidersRep/overlay/ for complete working example.
