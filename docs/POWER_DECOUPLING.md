# ⚡ Power Decoupling for the NeoPixel Matrix - LED-Garland-Anim

*Design decision record (hardware). Applies to the WS2812B 8x8 matrix on all supported boards.*

## Decision

**A 1000µF / 16V electrolytic capacitor across the 5V and GND inputs of the WS2812B matrix, as close to the matrix as possible, and a 470Ω resistor in series on the data line next to the first pixel, are documented as OPTIONAL protections.**

Status: **optional**. The original build ran without them for the whole of Christmas last year, with no reported fault. They are cheap insurance (about $1) against a class of hard-to-diagnose faults, worth adding if you see resets or flicker, if you raise the brightness, or if you change the supply or the wiring.

---

## 1. Why a WS2812B matrix draws "spiky" current

| Figure | Value |
|--------|-------|
| Current per pixel at full white | ~60mA (3 x 20mA) |
| Whole matrix, full white | 64 x 60mA = **3.84A** |
| Firmware default brightness | `matrixBrightness = 128` (~50%), i.e. up to ~1.9A worst case |
| Typical animation (a few lit pixels, mostly colored) | 0.1 to 0.8A |

The important point is not the average, it is the **variation**. An animation such as `FIREWORKS`, `CONFETTI`, `DISCO_BALL` or `PRESENTS` goes from almost black to many bright pixels from one frame to the next. Each pixel switches its LED drivers on within microseconds when the new frame is latched (the `show()` call), so the load jumps by hundreds of mA to several A in a very short time. A PWM-driven pixel also draws its current in pulses at ~400Hz, so even a static image has a pulsed load.

## 2. What the supply sees: the problem

A power supply and its wires cannot answer instantly:

1. **Wire inductance and resistance.** Dupont jumpers and breadboard rails are thin and long. A sudden current step `di` produces a voltage dip `V = R x I + L x di/dt`. With 0.3Ω of jumper and breadboard resistance, a 2A load already costs 0.6V before any transient.
2. **Regulation delay.** A switching supply's control loop reacts in hundreds of microseconds to milliseconds. During that time the output is held up only by its own output capacitor, which is not designed for a 3A step.
3. **Shared rail.** The ESP32 board, the TB6612FNG, the PIR sensor and the matrix are all fed from the same 5V. A dip caused by the matrix is therefore seen by everything else.

The result is a **short voltage dip (and ringing) on the 5V rail every time the picture changes a lot**.

## 3. What decoupling does

A capacitor is a tiny local energy tank. When the matrix suddenly asks for current, the capacitor delivers it from a few millimeters away, so the current does not have to travel through the long wires, and the supply has time to catch up. When the load drops, the capacitor absorbs the surplus.

Order of magnitude: the voltage drop while the capacitor supplies a current step is

```
ΔV = I x Δt / C
```

For a 1A step lasting 100µs with C = 1000µF: ΔV = 1 x 0.0001 / 0.001 = **0.1V**. With no capacitor the same step produces dips of 0.5V or more over the wiring alone.

Two levels of decoupling are involved, and they do different jobs:

| Part | Role | Where |
|------|------|-------|
| **1000µF electrolytic (bulk)** | Supplies the big, relatively slow steps (µs to ms), flattens the 400Hz PWM pulses | Across 5V/GND at the matrix input |
| **100nF ceramic (high frequency)** | Absorbs very fast edges (ns to µs). Most WS2812B matrix boards already include one per pixel, so nothing to add on the matrix | Already on the matrix. Add one close to the ESP32 board if it resets despite the bulk capacitor |

### Data line resistor

The 470Ω resistor on the data line (listed in the optional parts, now recommended) limits the current spike into the first pixel's data input at power-up and damps ringing on the signal edge. Place it close to the **matrix** end of the wire (the first pixel), not near the ESP32. Any value from 300Ω to 500Ω is accepted by the usual guidelines.

## 4. Consequences of not fitting a capacitor

None of these is guaranteed. A short, thick wiring and a good 5A supply may hide the problem, especially at brightness 128. They become likely with breadboard wiring, thin jumpers, a cheap supply or a high brightness setting.

| Symptom | Mechanism |
|---------|-----------|
| **ESP32 random resets / reboot loops** | The 5V dip passes through the board's 3.3V regulator (a typical AMS1117 needs ~4.4V at its input to stay in regulation). Below that, 3.3V sags, the ESP32 brownout detector fires and the chip reboots. Typically seen when an animation with many bright pixels starts, or when WiFi transmits at the same time (WiFi adds ~300mA peaks). |
| **WiFi drops, web UI unreachable, MQTT/OTA failures** | The radio is the first thing to suffer from a low or noisy 3.3V rail. An OTA update interrupted by a brownout can leave the device needing a USB re-flash. |
| **Flicker, wrong colors, first pixels glitching** | The data signal is referenced to GND. If GND bounces by a few hundred mV under load, the 3.3V data signal is no longer read cleanly by the WS2812B (its high threshold is already 0.7 x VDD = ~3.5V at 5V, so there is almost no margin with a 3.3V ESP32 output). |
| **Brightness "breathing" or dimming at the end of the matrix** | Voltage droop along the supply path. As the voltage falls, the blue and green dies (higher forward voltage) lose brightness first, so whites drift toward red/orange. |
| **Garland (TB6612FNG) flicker** | The garland and the matrix share 5V. Matrix load steps modulate the garland brightness. |
| **PIR false triggers** | The HC-SR501 is very sensitive to supply noise and often fires when the rail is noisy. This can start animations by itself, which then creates more load steps. |
| **Power-up stress on the first pixel** | Connecting a live 5V to a discharged, uncoupled matrix produces a voltage spike that can damage the first pixel's data input. Adafruit's NeoPixel guidelines list this as the main reason for the capacitor. |
| **Reduced component life** | Repeated spikes and brownouts stress the supply, the regulator and the pixels. Unlikely to cause immediate failure, but a plausible slow cause of dead pixels. |

## 5. What a capacitor does not fix

- An **undersized supply** (less than the peak current you ask for). Keep 3A minimum, 5A ideal, or lower the brightness.
- **Thin or very long wires.** Use short, thick wires (22AWG or better) for the 5V and GND to the matrix, and a direct link rather than going through the breadboard rails if possible.
- **A 3.3V to 5V logic level problem.** If the first pixels glitch while the supply is perfectly stable, add a level shifter (74AHCT125 or 74HCT245) between the ESP32 data pin and the matrix.

## 6. Installation checklist

- [ ] Capacitor **1000µF, voltage rating 10V or more** (16V recommended), low ESR preferred
- [ ] **Polarity**: the stripe on the can marks the negative leg, to GND. Reversing it can make it burst
- [ ] Connect it **directly at the matrix 5V/GND input**, with the shortest leads possible, not at the power supply end
- [ ] Connect **GND first** and power last when wiring the matrix, and avoid plugging the matrix on an already powered rail
- [ ] 470Ω resistor in series on the data line, close to the matrix
- [ ] Optional: a second capacitor (470 to 1000µF) at the ESP32 5V input if the board still resets
- [ ] Optional safety: keep the firmware brightness at or below 128 unless the supply is rated 5A

## 7. Optional protections summary

| Protection | Useful? | Comment |
|------------|---------|---------|
| 1000µF capacitor at the matrix input | Yes | Very recommended if you see symptoms (section 4) |
| 330-470Ω resistor on DATA | Yes | Classic Adafruit recommendation |
| Series diode on the matrix 5V feed | Optional | Lowers the matrix VDD so the 3.3V data signal is read more reliably (see below) |
| Reverse-polarity diode | Optional | Only as an extra safety |

### Series diode on the 5V feed (optional)

The WS2812B reads a logic high above 0.7 x VDD, about 3.5V at 5V, which a 3.3V ESP32 output barely reaches. A diode in series with the matrix 5V drops 0.3 to 0.7V, so the matrix sees about 4.3 to 4.7V and its threshold falls to about 3.0 to 3.3V. The 3.3V data is then read with more margin.

- It is often described as "a diode on the first pixel". **On an 8x8 matrix you cannot isolate the first pixel**, because all 64 pixels share the same rail, so the diode goes on the whole matrix feed.
- **Use a 3A Schottky** (1N5822 or SB360). A 1N4001 is rated 1A and drops more than 1V at high current, which is unsuitable here.
- Place the **1000µF capacitor after the diode**, on the matrix side.
- Keep the ESP32 and the other loads fed **before** the diode, from the unmodified 5V.
- The drop changes with current, so this is a workaround. A level shifter (74AHCT125 or 74HCT245) is the proper fix and costs about the same.

### Reverse-polarity diode (optional)

Only if you want extra safety against a swapped supply. A series Schottky does both jobs (reverse protection and lowering the VDD threshold), at the cost of 0.3 to 0.4V on the matrix rail.

## Related

- [HARDWARE_GUIDE.md](HARDWARE_GUIDE.md): component list and power supply sizing
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md): reset and flicker diagnostics
- [PIN_MAPPING.md](PIN_MAPPING.md): matrix data pin per board
