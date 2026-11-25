# Release Notes v5.6.5 (2026-11-25)

**Valid from version 5.6.5**

Documentation-only release. No firmware behavior change.

## 📚 Documentation

### New: NeoPixel Matrix Power Decoupling
New hardware note [POWER_DECOUPLING.md](POWER_DECOUPLING.md) ([français](POWER_DECOUPLING_FR.md)) covering:
- Why a WS2812B 8x8 matrix draws spiky current (up to 3.84A at full white, ~1.9A at the default brightness of 128)
- What the 5V rail sees and what a 1000µF capacitor does about it
- Consequences without it: ESP32 brownout resets, WiFi/OTA drops, flicker and glitched first pixels, garland flicker, PIR false triggers
- What a capacitor does not fix (undersized supply, thin wires, 3.3V to 5V logic level)
- Installation checklist

### Optional protections
All of these are **optional**: the original build ran without them for the whole of Christmas last year.

| Protection | Status |
|------------|--------|
| 1000µF capacitor at the matrix input | Optional, recommended if resets or flicker appear |
| 330-470Ω resistor on DATA | Optional |
| 3A Schottky diode in series on the matrix 5V feed | Optional (lowers the VDD threshold for the 3.3V data signal) |
| Reverse-polarity diode | Optional extra safety |

### Hardware Guide
[HARDWARE_GUIDE.md](HARDWARE_GUIDE.md) / [HARDWARE_GUIDE_FR.md](HARDWARE_GUIDE_FR.md): capacitor and resistor documented as optional, Schottky diode added to the optional parts, links to the new note.

---

## 🚀 Installation

```bash
git pull origin main
git checkout v5.6.5
pio run -e <your_board> -t upload
```

Since the firmware is unchanged, flashing is only needed to update the displayed version number.

---

**Previous Release**: [v5.6.4](RELEASE_v5.6.4.md)  
**Full Changelog**: [CHANGELOG.md](../CHANGELOG.md)
