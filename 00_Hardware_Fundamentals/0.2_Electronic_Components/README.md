# 0.2 Electronic Components

## Overview

Electronic components are the building blocks of circuits. As a software engineer, you need to understand **what each component does** and **why it's placed in the circuit**.

---

## Resistors (저항)

### Basic Function
- Limits current flow
- Creates voltage drop
- Dissipates power as heat

### Symbol
```
 ───[  ]───  (US)
 ───[ZZZ]───  (International)
```

---

## Pull-up / Pull-down Resistors (★ CRITICAL)

### Why They're Needed

**Problem: Floating Input**
```
    MCU GPIO Input (floating)
        |
        X  ← Nothing connected = undefined voltage!
        
Reads random values: sometimes 0, sometimes 1
```

### Pull-up Resistor

**Pulls input to HIGH when nothing drives it**

```
    VDD (3.3V)
     |
    [10kΩ] ← Pull-up resistor
     |
    GPIO ────[Switch]──── GND
     |
```

**States:**
- Switch open: GPIO reads HIGH (pulled to VDD)
- Switch closed: GPIO reads LOW (pulled to GND)

**Common Uses:**
- I2C bus lines (SDA, SCL) - **REQUIRED!**
- Button inputs
- Open-drain outputs

### Pull-down Resistor

**Pulls input to LOW when nothing drives it**

```
    GPIO ────[Switch]──── VDD
     |
    [10kΩ] ← Pull-down resistor
     |
    GND
```

**States:**
- Switch open: GPIO reads LOW (pulled to GND)
- Switch closed: GPIO reads HIGH (pulled to VDD)

### Resistor Value Selection

```
Typical values: 1kΩ - 100kΩ

Strong pull (faster, more power):    1kΩ - 4.7kΩ
Standard:                             10kΩ
Weak pull (slower, less power):      47kΩ - 100kΩ

Trade-off:
- Smaller R → faster switching, more current
- Larger R → slower switching, less current
```

---

## Current Limiting Resistors

### LED Protection (★ COMMON)

**LEDs will burn out without current limiting!**

```
Calculation:
V_supply = 5V
V_led = 2.0V (red LED typical)
I_led = 20mA (desired brightness)

R = (V_supply - V_led) / I_led
R = (5V - 2.0V) / 0.020A = 150Ω

Power: P = I² × R = (0.02)² × 150 = 0.06W
→ Use 1/4W (0.25W) resistor
```

**LED Forward Voltages:**
- Red: ~2.0V
- Green: ~2.2V
- Blue/White: ~3.0-3.5V
- IR: ~1.2V

---

## Voltage Divider

**Scales down voltage**

```
         V_in
          |
         [R1]
          |
          +──── V_out = V_in × R2/(R1+R2)
          |
         [R2]
          |
         GND
```

### Applications

**1. ADC Input Scaling**
```
Problem: 12V battery voltage, 3.3V ADC max

Solution: Scale down by 1/4
R1 = 30kΩ
R2 = 10kΩ
V_out = 12V × (10kΩ/40kΩ) = 3.0V ✓
```

**2. Logic Level Conversion** (crude, better use level shifter)
```
5V → 3.3V
R1 = 1kΩ
R2 = 2kΩ
V_out = 5V × (2kΩ/3kΩ) = 3.33V
```

⚠️ **Disadvantages:**
- Wastes power (continuous current)
- Output impedance affects accuracy
- Cannot drive significant load

---

## Capacitors (커패시터, 콘덴서)

### Basic Function
- Stores electrical energy
- Blocks DC, passes AC
- Smooths voltage
- Creates timing circuits

### Symbol
```
 ───| |───  (non-polarized)
 ───|+|───  (polarized/electrolytic)
```

---

## Decoupling / Bypass Capacitors (★ CRITICAL)

### Purpose: Filter Power Supply Noise

**Problem without decoupling:**
```
Power Supply ────[long trace]──── MCU
                                   |
                              [sudden current draw]
                                   |
→ Voltage dips, MCU crashes!
```

**Solution: Add capacitor close to IC**
```
    VDD ─────────────┬──── MCU VDD pin
                     |
                    === 100nF (0.1µF)
                     |
    GND ─────────────┴──── MCU GND pin
```

### Typical Values

**Every IC power pin needs decoupling!**

```
Ceramic capacitors (close to IC):
- 100nF (0.1µF) - standard for digital ICs
- 10nF (0.01µF) - high-speed ICs

Electrolytic capacitors (power input):
- 10µF - 100µF for bulk storage
```

### Placement Rules

✅ **DO:**
- Place 100nF ceramic cap **as close as possible** to each VDD pin
- Place 10µF-100µF bulk cap at power entry
- Use short traces

❌ **DON'T:**
- Place caps far from IC
- Share one cap between multiple ICs
- Use only large capacitors (need both bulk + decoupling)

---

## Timing Capacitors (RC Delay)

### RC Time Constant

```
τ (tau) = R × C

Example:
R = 10kΩ
C = 1µF
τ = 10kΩ × 1µF = 10ms

Voltage charges to 63% in one τ
Fully charged (~99%) in 5τ = 50ms
```

### Applications

**1. Button Debouncing**
```
    VDD
     |
    [10kΩ]
     |
    GPIO ───[100nF]─── GND
     |
   [Button to GND]

Debounce time ≈ R×C = 10kΩ × 100nF = 1ms
```

**2. Reset Circuit**
```
    VDD
     |
    [10kΩ]
     |
    RST ───[10µF]─── GND

Power-on delay ≈ 5 × R×C = 5 × 100ms = 500ms
```

---

## Inductors (인덕터, 코일)

### Basic Function
- Opposes **change** in current
- Stores energy in magnetic field
- Used in switching regulators

### Symbol
```
 ───∿∿∿───
```

### LC Filter (Noise Filtering)

```
VDD_in ───∿∿∿─┬─── VDD_clean
        inductor |
                ===
                 |
                GND
```

**Applications:**
- Power supply filtering
- RF circuits
- Buck/boost converters

---

## Back EMF and Flyback Diode (★ MOTOR CONTROL)

### Problem: Inductive Kickback

**When motor/relay stops, inductor voltage spikes!**

```
         +12V
          |
       [Motor/Relay]  ← Inductor
          |
     [MOSFET] ← We turn this OFF
          |
         GND

When MOSFET turns OFF:
→ Current can't stop instantly
→ Voltage spikes to hundreds of volts!
→ MOSFET dies! ⚡💀
```

### Solution: Flyback Diode

```
         +12V
          |
      [  Motor  ]
      |        |
      |       ▼| ← Flyback diode (reverse biased)
      |      [|─┐
      |        |
     [MOSFET]  |
      |        |
     GND ──────┘

Diode provides path for current when MOSFET off
```

**Diode Selection:**
- Fast recovery diode (e.g., 1N4148 for small loads)
- Schottky diode for faster switching
- Current rating ≥ motor current

---

## Diodes (다이오드)

### Symbol and Operation

```
    Anode  Cathode
      |▶─────|
      
Current flows: Anode → Cathode (when forward biased)
Forward voltage drop: ~0.7V (standard), ~0.3V (Schottky)
```

---

## Diode Types

### 1. Rectifier Diode (정류 다이오드)

**One-way valve for current**

```
Applications:
- AC to DC conversion
- Reverse polarity protection
```

### 2. Zener Diode (제너 다이오드)

**Maintains constant voltage when reverse biased**

```
Symbol:
    |▶─|─
      ─|
      
Applications:
- Voltage regulation (crude)
- Overvoltage protection
```

**Example: 5V Overvoltage Clamp**
```
Signal ───[1kΩ]───┬──── To MCU
                  |
                 ─|─ 5.1V Zener
                  |
                 GND
                 
If signal > 5.1V, Zener conducts → clamps voltage
```

### 3. Schottky Diode (쇼트키 다이오드)

**Fast switching, low forward voltage drop**

```
Forward voltage: ~0.3V (vs 0.7V for standard)

Applications:
- High-frequency rectification
- Flyback diodes for fast motors
- Reverse polarity protection (less power loss)
```

### 4. TVS Diode (Transient Voltage Suppressor)

**ESD (정전기) protection**

```
Signal ───────┬──── To MCU
             ─|─ TVS diode
              |
             GND
             
Clamps voltage spikes (e.g., from static discharge)
```

---

## Transistors (트랜지스터)

### As Electronic Switches

**Software engineers mainly use transistors as switches**

---

## BJT (Bipolar Junction Transistor)

### NPN Transistor

```
        C (Collector)
        |
        |▶ 
     B ─|  (Base)
        |▷
        |
        E (Emitter)
```

**Operation as Switch:**
```
     VCC (+5V)
      |
    [Load/LED]
      |
     C|
  B ──|▶  NPN
     E|
      |
     GND

When Base > 0.7V: Transistor ON (C to E conducts)
When Base = 0V:   Transistor OFF (C to E open)
```

**Base Resistor Calculation:**
```
I_load = 100mA
h_FE (gain) = 100
I_base = I_load / h_FE = 1mA

V_gpio = 3.3V
V_be = 0.7V
R_base = (V_gpio - V_be) / I_base
R_base = (3.3V - 0.7V) / 0.001A = 2.6kΩ
→ Use 2.7kΩ or 3.3kΩ
```

---

## MOSFET (★ PREFERRED FOR EMBEDDED)

### Why MOSFETs are Better than BJTs

✅ **Voltage-controlled (not current)**
- GPIO can drive directly (with appropriate resistor)
- No continuous base current

✅ **Lower ON resistance (RDS_on)**
- Less power dissipation
- Can handle higher currents

✅ **Faster switching**

### N-Channel MOSFET

```
        D (Drain)
        |
     G ─|─┤ (Gate)
        |
        S (Source)
```

**Low-Side Switch (Most Common):**
```
     VCC (+12V)
      |
    [Load/Motor]
      |
     D|
  G ──|─┤ N-CH MOSFET
     S|
      |
     GND

When Gate > Threshold (~2-4V): MOSFET ON
When Gate = 0V:                MOSFET OFF
```

**Gate Resistor (pull-down):**
```
     MCU GPIO
      |
     [10kΩ] ← Prevents floating gate
      |
  G ──|─┤ MOSFET
     S|
      |
     GND
```

### P-Channel MOSFET

```
High-Side Switch:
     VCC (+12V)
      |
     S|
  G ──|─┤ P-CH MOSFET (inverted!)
     D|
      |
    [Load]
      |
     GND

When Gate < (VCC - Vth): MOSFET ON
When Gate = VCC:         MOSFET OFF
```

⚠️ **P-Channel is inverted: LOW turns ON!**

---

## Switch / Relay

### Mechanical Relay

```
Coil ─────┬───── +12V
          |
        (coil)
          |
Coil ─────┴───── GND (via transistor)

     NC ○──╮
            ├── COM ○
     NO ○──╯

NC = Normally Closed
NO = Normally Open
COM = Common
```

**When coil energized: COM connects to NO**

### Solid State Relay (SSR)

- Electronic switching (no moving parts)
- Faster, longer life
- More expensive

---

## Key Takeaways

| Component | Primary Use | Critical For |
|-----------|-------------|--------------|
| Pull-up/down Resistor | Define logic levels | I2C, buttons, inputs |
| Decoupling Capacitor | Power noise filtering | **Every IC!** |
| Flyback Diode | Inductive kickback protection | Motors, relays |
| MOSFET | Electronic switch | High-power control |
| Zener/TVS Diode | Voltage protection | ESD, overvoltage |

---

## Practical Exercises

1. **Design button circuit** with proper pull-up
2. **Calculate decoupling cap values** for an MCU
3. **Add flyback diode** to relay circuit
4. **Select MOSFET** for 2A motor at 12V
5. **Design ESD protection** for USB data lines

---

**Next:** [0.3 Digital Logic & Interface](../0.3_Digital_Logic/README.md)
