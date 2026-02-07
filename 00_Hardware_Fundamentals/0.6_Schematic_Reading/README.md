# 0.6 Schematic Reading & Tools

## Overview

Reading circuit schematics is an **essential skill** for embedded software engineers. You don't need to design circuits, but you must understand them to:
- Debug hardware-software interface issues
- Understand GPIO configurations
- Communicate with hardware engineers
- Interpret datasheets

---

## Schematic Symbols (Common Components)

### Power and Ground

```
VDD, VCC, V+         Positive power supply
     |
     ↓
    ─┴─  or  ───●───
    
GND, VSS, V-         Ground (0V reference)
    ─┬─  or  ───▼───
     ⊥
```

**Multiple Power Rails:**
```
+12V    +5V    +3.3V    +1.8V
 |       |       |        |
```

**Analog vs Digital Ground:**
```
AGND - Analog ground (cleaner, noise-sensitive)
DGND - Digital ground (noisy)

Often connected at ONE point only (star ground)
```

---

## Component Symbols Reference

### Passive Components

```
Resistor:          ───[  ]───  or  ───[ZZZ]───

Capacitor:         ───| |───     (non-polarized)
                   ───|+|───     (polarized)

Inductor:          ───∿∿∿───

Potentiometer:     ───[/]───
                       |
```

### Semiconductors

```
Diode:             ───|▶|───
                   Anode  Cathode

LED:               ───|▶|───
                      ↑|

Zener Diode:       ───|▶─|─
                        ─|

NPN BJT:               C
                       |
                    B─|▷
                       E

N-CH MOSFET:           D
                       |
                    G─|─┤
                       S

Optocoupler:       LED |▶| Photo-transistor
                      ─╫─
```

### Logic and ICs

```
IC (DIP):           ┌────┐
                  1─|    |─8
                  2─|    |─7
                  3─|    |─6
                  4─|    |─5
                    └────┘

Microcontroller:   ┌─────────┐
                 1─| PA0  VDD|─64
                 2─| PA1  GND|─63
                   |   MCU   |
                63─| PB5  RST|─2
                64─| PB6  VDD|─1
                   └─────────┘
```

---

## Reading Connection Conventions

### Wires and Connections

**Connected (dot at intersection):**
```
    ───┬───
       │
       │
       ●  ← Dot = connection
       │
```

**Not Connected (crossing lines):**
```
    ───┼───  or  ───┐
       │             └───
                   
No dot = no connection
```

**Net Labels (same name = connected):**
```
Circuit on page 1:
    GPIO_PA5 ──────●

Circuit on page 3:
    ●────── GPIO_PA5
    
Same label = connected electrically
```

---

## Reading a Real Schematic

### Example: MCU Power Supply Section

```
+5V_IN ─┬────[10µF]────┬────────────┬──── +3.3V_OUT
        │              │            │
       ┌┴┐           ┌─┴─┐        ┌─┴─┐
       │ │ 470Ω      │ │ │ 100nF  │ │ │ 100nF
       │ │           │ │ │        │ │ │
       └┬┘           └─┬─┘        └─┬─┘
        │              │            │
    ┌───┴───┐          │            │
  1─┤VIN VOUT├─2───────┴────────────┴────
  3─┤GND  EN ├─4
    └───────┘
    LD1117-3.3

GND ────┴───────────┴─────────────┴────
```

**What you should see:**
1. **Input filtering:** 10µF capacitor on +5V input
2. **Voltage regulator:** LD1117-3.3 (LDO, 3.3V output)
3. **Current limiting:** 470Ω resistor on input (optional, for protection)
4. **Decoupling caps:** 100nF ceramics on output (x2 for redundancy)
5. **Enable pin (EN):** Pulled high (always enabled)

---

## Common Circuit Patterns

### Pattern 1: Reset Circuit

```
+3.3V ────┬────────────┬──── VDD
          │            │
         ┌┴┐         ┌─┴─┐
         │ │ 10kΩ    │   │ 100nF
         │ │         │   │
         └┬┘         └─┬─┘
          │            │
          ├────────────┴──── RST (MCU pin)
          │            
       [Button]        
          │            
GND ──────┴────────────────── GND
```

**Function:**
- Pull-up resistor keeps RST high (normal operation)
- Button press pulls RST low (reset MCU)
- 100nF cap debounces button

---

### Pattern 2: Crystal Oscillator

```
        OSC_IN               OSC_OUT
MCU ──────┤                     ├────── MCU
          │                     │
         ┌┴┐                   ┌┴┐
         │ │ 22pF              │ │ 22pF
         │ │                   │ │
         └┬┘                   └┬┘
          │     [Crystal]       │
          ├──────∿∿∿∿∿∿────────┤
          │      8MHz           │
GND ──────┴─────────────────────┴──── GND
```

**Function:**
- Crystal provides accurate clock frequency
- Load capacitors (22pF) tune frequency
- Values specified in MCU datasheet

---

### Pattern 3: I2C Pull-ups

```
+3.3V ─┬────────────────────┬────────
       │                    │
      ┌┴┐                  ┌┴┐
      │ │ 4.7kΩ (R1)       │ │ 4.7kΩ (R2)
      │ │                  │ │
      └┬┘                  └┬┘
       │                    │
       ├─── SDA ────────────●─────── (to devices)
       │
       └─── SCL ────────────●─────── (to devices)
```

**Function:**
- I2C requires pull-up resistors on SDA and SCL
- Open-drain outputs need pull-ups
- Typical values: 2.2kΩ - 10kΩ

---

### Pattern 4: Motor Driver (H-Bridge)

```
              +12V
               │
         Q1 ┌─┴─┐ Q2
      ──────┤   ├──────
            └─┬─┘
              │
           [Motor]
              │
         Q3 ┌─┴─┐ Q4
      ──────┤   ├──────
            └─┬─┘
              │
             GND

Q1, Q4 ON → Motor forward
Q2, Q3 ON → Motor reverse
All OFF   → Motor brake (or coast)
```

---

## Interpreting Datasheets

### Essential Sections to Read

**1. Pin Configuration**
```
Look for:
- Pin number
- Pin name (PA0, PB5, etc.)
- Alternate functions
- Default state (pull-up/down/floating)
```

**2. Electrical Characteristics**
```
Key parameters:
- VDD range (e.g., 2.0V - 3.6V)
- I_max per pin (e.g., 25mA)
- VIH (input high voltage): minimum for logic '1'
- VIL (input low voltage): maximum for logic '0'
- VOH (output high voltage)
- VOL (output low voltage)
```

**3. Absolute Maximum Ratings**
```
⚠️ NEVER exceed these!
- Maximum voltage on any pin
- Maximum current
- Storage temperature
- ESD sensitivity
```

### Example: Reading GPIO Specifications

```
From STM32F4 datasheet:

VIH (input high):  2.0V (when VDD=3.3V)
VIL (input low):   0.8V

This means:
- Voltage > 2.0V → reads as '1'
- Voltage < 0.8V → reads as '0'
- 0.8V to 2.0V → undefined! (noise margin)

Maximum output current: 25mA per pin
→ Can drive LED with resistor
→ Cannot drive motor directly!
```

---

## Measurement Tools

### Multimeter (멀티미터)

**Basic Measurements:**

**1. Voltage (DC)**
```
Settings: DC V (⎓)
Probe: Red to signal, Black to GND
Measures: Voltage relative to GND
```

**2. Continuity (통전 테스트)**
```
Settings: Continuity (🔊 symbol)
Probe: Touch both ends of wire
Beep → connected (short)
Silent → open (broken)
```

**3. Resistance**
```
Settings: Ω (Ohms)
Probe: Across resistor (POWER OFF!)
Reads: Resistance value
```

---

### Oscilloscope (오실로스코프)

**What it shows:** Voltage over time (waveform)

**Common Uses:**
1. **Verify PWM signal**
   - Check duty cycle
   - Measure frequency
   - Observe rise/fall times

2. **Debug UART communication**
   - Verify baud rate
   - Check signal integrity
   - Detect glitches

3. **Power supply analysis**
   - Measure ripple voltage
   - Check startup behavior
   - Verify decoupling effectiveness

**Key Controls:**
- **Vertical scale:** V/div (voltage per division)
- **Horizontal scale:** Time/div (time per division)
- **Trigger:** When to start capturing (edge, level)

---

### Logic Analyzer (로직 분석기)

**What it shows:** Digital signals (0/1) for multiple channels

**Advantages over Oscilloscope:**
- Many channels (8, 16, 32+)
- Protocol decoding (I2C, SPI, UART, CAN)
- Longer capture time

**Common Uses:**
1. **Debug I2C communication**
   - Decode address and data bytes
   - Check ACK/NACK
   - Verify timing

2. **Analyze SPI signals**
   - Decode MOSI/MISO data
   - Check clock polarity/phase
   - Measure CS timing

3. **Verify GPIO timing**
   - Measure pulse widths
   - Check setup/hold times
   - Find timing violations

---

## Common Schematic Reading Mistakes

### ❌ Mistake 1: Ignoring Pull-ups

```
Schematic shows GPIO connected to button without pull-up

Developer: "Why does button read random values?"

→ Need to enable internal pull-up in code
   or add external resistor!
```

### ❌ Mistake 2: Wrong Logic Level

```
3.3V MCU → 5V sensor (direct connection)

Developer: "Why is the sensor output damaging GPIO?"

→ Need level shifter or voltage divider
```

### ❌ Mistake 3: Exceeding Current Limits

```
GPIO → LED (no resistor) → GND

Developer: "Why did the GPIO pin die?"

→ GPIO max current (25mA) exceeded
   Need current-limiting resistor!
```

---

## Practical Exercises

### Exercise 1: Identify Components
Given a schematic, identify:
- All power supply rails
- Decoupling capacitors
- Reset circuit
- Crystal oscillator
- GPIO connections

### Exercise 2: Debug from Schematic
```
Problem: I2C device not responding

Check schematic:
1. Are pull-up resistors present?
2. Correct voltage level (3.3V or 5V)?
3. SDA/SCL connected to correct pins?
4. Power supply properly decoupled?
```

### Exercise 3: Calculate Values
```
Given: LED circuit in schematic
- Supply voltage
- LED forward voltage
- Desired current

Calculate: Resistor value and power rating
```

---

## Tips for Reading Schematics Efficiently

1. **Start with power:** Trace VDD and GND first
2. **Follow signal flow:** From input to MCU to output
3. **Check reference designators:** R1, C1, U1 (matches BOM)
4. **Read net labels:** Same name = connected
5. **Don't panic:** Start with one section at a time
6. **Ask why:** Every component has a purpose

---

## Resources

### Online Simulators
- **Falstad Circuit Simulator:** Visual, interactive
- **LTSpice:** Professional, free from Analog Devices
- **EveryCircuit:** Mobile app, good for learning

### Schematic Software
- **KiCad:** Free, open-source PCB design
- **EasyEDA:** Online, free
- **Altium Designer:** Professional (expensive)
- **Eagle:** Hobbyist-friendly

### Learning Resources
- EEVblog (YouTube): Dave Jones' circuit analysis
- AddOhms (YouTube): Electronics tutorials
- SparkFun tutorials: Beginner-friendly guides
- Component datasheets: Always start here!

---

## Summary Checklist

After this section, you should be able to:
- ✅ Read and understand basic circuit schematics
- ✅ Identify common electronic components by symbol
- ✅ Interpret net labels and connections
- ✅ Recognize common circuit patterns (reset, oscillator, etc.)
- ✅ Extract key information from datasheets
- ✅ Use multimeter for basic measurements
- ✅ Understand oscilloscope and logic analyzer purpose
- ✅ Debug hardware issues using schematics

---

**Congratulations!** You've completed Hardware Fundamentals (Layer 0).

**Next Step:** [01 Embedded Fundamentals](../../01_Embedded_Fundamentals/README.md)
