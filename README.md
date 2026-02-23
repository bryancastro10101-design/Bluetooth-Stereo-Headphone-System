# 🎧 Bluetooth Stereo Headphone System
**EE 445L Embedded System Design Lab — Final Project | The University of Texas at Austin**

A fully functional wireless stereo audio system built from scratch using an ESP32 and TM4C123 microcontroller, featuring real-time DSP, dual TFT displays, and a custom PCB designed in KiCad.

---

## 📌 Project Overview

This project demonstrates end-to-end embedded system design — from hardware architecture and PCB layout to firmware development and digital signal processing. Audio is streamed wirelessly from a PC via Bluetooth (A2DP protocol), decoded by an ESP32, and transmitted over SPI to a TM4C123 microcontroller for processing and playback through dual speakers.

---

## 🏗️ System Architecture

```
[PC / Spotify]
     │
     │  Bluetooth A2DP
     ▼
[ESP32 DevKitC]  ──── SPI ────►  [TM4C123GH6PM]
                                       │
                          ┌────────────┴────────────┐
                          │                         │
                     [TLV5616 DAC]           [TLV5616 DAC]
                          │                         │
                    [MCP34119 OPA]           [MCP34119 OPA]
                          │                         │
                      [Speaker L]             [Speaker R]
                          
         [ST7735R LCD ×2]  ◄── SPI ── TM4C123
         [Volume Buttons ×4]
```

**Communication Protocols:**
- **Bluetooth A2DP** — Wireless audio streaming from PC to ESP32
- **SPI** — ESP32 → TM4C123 audio data transfer; TM4C123 → DACs and LCD screens
- **I2C** — ESP32 ↔ TM4C123 control/status channel

---

## ⚙️ Hardware Components

| Component | Part | Qty | Role |
|---|---|---|---|
| Microcontroller | TM4C123GH6PM | 1 | Main audio processor |
| Wireless MCU | ESP32-DevKitC-V4 | 1 | Bluetooth A2DP receiver |
| TFT Display | ST7735R | 2 | Real-time audio level UI |
| DAC | TLV5616 | 2 | Digital-to-analog conversion |
| Op-Amp | MCP34119P | 2 | Audio amplification |
| Buck Regulator | MPM3610GQV | 2 | 9V → 5V and 9V → 3.3V |
| Voltage Reference | LM4041CILPR | 1 | Stable DAC reference |

---

## 🔋 Power Budget

| Rail | Load Breakdown | Range |
|---|---|---|
| **3.3V** | TM4C123 + 2× ST7735R + ESP32 + regulators + Vref | 231 mA – 321 mA |
| **5V** | 2× MCP34119 + 2× TLV5616 | ~2.32 mA |

Both rails are supplied by MPM3610GQV switching regulators from a 9V source, chosen for their high efficiency and small footprint.

---

## 💻 Firmware Design

### ESP32 — Bluetooth Receiver
Receives audio data over Bluetooth A2DP and streams it to the TM4C123 via SPI as fast as data is available.

```c
// ESP32 pseudocode
int main() {
    while (true) {
        while (SoundDataReady) {
            SPIOut(SoundData);
        }
    }
    return 0;
}
```

### TM4C123 — Audio Processor
Acts as the SPI slave, processes incoming audio data, and distributes it to the two DAC channels and both LCD displays.

**Module structure:**
```
dac.h       → DAC_Init(), DAC_Output(uint16_t data)
slave.h     → Slave_Init(void (*handler)(uint8_t databyte))
speaker.h   → init_Speaker(), play_Sound(int soundData)
lcd.h       → init_LCD(), update_LCD(update_Params params)
```

---

## 🖥️ User Interface

- **2× ST7735R TFT displays** showing real-time audio level visualization that responds dynamically to volume
- **4 buttons** (2 per channel) for:
  - Volume control (up/down)
  - Equalization adjustment

---

## 📐 PCB Design

Designed in **KiCad** targeting **JLCPCB** manufacturing rules.

**Key design decisions:**
- Regulators placed close to the battery connector to minimize long power traces
- Speaker/Amp/DAC subcircuit routed once and mirrored for the stereo channel
- Booster pack headers spaced for TM4C123 LaunchPad compatibility
- Crystal oscillator isolated from high-speed signal traces

---

## 📊 Performance Targets

| Metric | Target |
|---|---|
| Frequency Response | 20 Hz – 20 kHz |
| SNR | ≥ 80 dB |
| Button Response Time | Measured via logic analyzer |
| Bluetooth Range | Up to ~10 m (Class 2) |

---

## 🛠️ Skills Demonstrated

- **Embedded C firmware** on ARM Cortex-M4 (TM4C123)
- **SPI/I2C protocol implementation** (master & slave)
- **Bluetooth A2DP** wireless audio protocol
- **Digital Signal Processing** — real-time audio equalization
- **PCB design** in KiCad (schematic capture, layout, DRC)
- **Analog circuit design** — DAC output stage with op-amp buffering
- **Power electronics** — switching regulator selection and layout
- **Hardware/software co-design** across two MCUs

---

## 📁 Repository Structure

```
├── firmware/
│   ├── esp32/          # Bluetooth A2DP receiver code
│   └── tm4c/           # Audio processor, DAC, LCD, SPI slave
├── hardware/
│   ├── schematic/      # KiCad schematic files
│   ├── pcb/            # KiCad PCB layout files
│   └── datasheets/     # Component datasheets
├── docs/
│   ├── system_diagram.pdf
│   ├── requirements_document.pdf
│   └── power_calculations.txt
└── README.md
```

---

## 🎓 Course Context

**EE 445L — Embedded Systems Design Lab**  
Department of Electrical and Computer Engineering  
The University of Texas at Austin

This course covers real-time embedded system design, hardware/software integration, and PCB prototyping with an emphasis on the full engineering design cycle from requirements through implementation and testing.

---

---

## 📝 Lessons Learned

The board flashed successfully with all components soldered — until the buck converters were added. Once the MPM3610GQV regulators were populated, the board could no longer be programmed via JTAG.

After consulting with other engineers, the team's leading hypothesis is **switching noise crosstalk from the buck converters into the JTAG lines**. The regulators were placed in relative proximity to the JTAG header on a 2-layer PCB, which lacks the dedicated ground and power planes of a 4-layer design. Without those inner planes acting as a Faraday shield and low-impedance return path, the high-frequency switching transients generated by the regulators likely coupled into the JTAG data lines and disrupted the debug interface.

The audio side had its own surprise as well. One of the external ADC converters was producing noticeably poor sound quality that we couldn't tune out. Rather than spin a new board, the team jumped a wire on the back of the PCB to bypass it and route the signal directly into the ESP32's onboard ADC instead — which performed considerably better. It wasn't pretty, but it worked, and the project stayed on schedule. A classic embedded engineering field fix.

**Key takeaways for future designs:**
- Move to a **4-layer stackup** (signal / ground / power / signal) to provide shielding and clean return paths — especially important when mixing switching regulators with sensitive digital interfaces
- Increase the **physical separation** between switching regulators and JTAG/SWD headers, and avoid routing debug lines near regulator switching nodes
- Add **bulk and bypass capacitance** close to the regulator output to suppress switching noise at the source
- Consider adding a **ferrite bead or LC filter** on the power rail feeding the microcontroller to further isolate debug circuitry from regulator noise

---

*Designed and implemented as part of the EE 445L final project. All schematics, firmware, and documentation produced independently.*
