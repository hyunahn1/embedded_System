# 0.4 Power Electronics

## Overview

Power electronics deals with converting and controlling electrical power. Essential for:
- Powering your embedded system from batteries or mains
- Motor control (EV, robotics, drones)
- Battery management systems (BMS)
- Efficient power delivery

---

## Voltage Regulation

### Why Regulators?

```
Problem: Voltage sources are not perfect
- Battery: 12.6V (full) → 10.5V (empty)
- Wall adapter: 9V ± 10% = 8.1V to 9.9V
- But MCU needs: 3.3V ± 5% = 3.135V to 3.465V

Solution: Voltage regulator
   Vin (variable) → [Regulator] → Vout (stable)
```

---

## LDO (Linear Regulator)

### How it Works

```
   Vin ──┬────[Variable Resistor]────┬─── Vout
         │    (controlled by IC)     │
         │                         [Load]
         │                           │
        [Control Circuit]            │
         │                           │
   GND ──┴───────────────────────────┴─── GND

Excess voltage dropped as heat:
P_heat = (Vin - Vout) × I_load
```

### Characteristics

```
✅ Pros:
- Simple (3 pins: Vin, Vout, GND)
- Low noise (clean output)
- No switching noise
- Cheap

❌ Cons:
- Inefficient (wastes energy as heat)
- Only step-down (Vout < Vin)
- Heats up with large Vin-Vout difference

Efficiency = Vout / Vin
Example: 12V → 3.3V = 3.3/12 = 27.5% 🔥
```

### Common LDO ICs

```
AMS1117-3.3: 3.3V, 1A, very cheap
LD1117-3.3:  3.3V, 800mA
LM7805:      5V, 1A (classic)
MIC5205:     3.3V, 150mA, low dropout
```

### When to Use LDO

✅ **Use LDO when:**
- Low noise critical (ADC reference, analog circuits)
- Small voltage drop (5V → 3.3V)
- Low current (<500mA)
- Simple design preferred

❌ **Don't use LDO when:**
- Large voltage drop (12V → 3.3V with high current)
- Battery powered (wastes energy)
- Need step-up (boost)

### Heat Calculation Example

```
Problem: Power 3.3V MCU from 12V battery
Current: 500mA

Power dissipated:
P = (Vin - Vout) × I
P = (12V - 3.3V) × 0.5A = 4.35W 🔥🔥🔥

Thermal resistance: 65°C/W (TO-220 package)
Temperature rise: 4.35W × 65°C/W = 283°C

→ HEATSINK REQUIRED!
→ Or use SMPS instead
```

---

## SMPS (Switching Regulator)

### How it Works (Buck Converter)

```
   Vin ──┬───[Switch]───┬───[L]───┬─── Vout
         │   (MOSFET)   │  Coil   │
         │              │         ===
         │            [Diode]     │
         │              │         │
   GND ──┴──────────────┴─────────┴─── GND

Operation:
1. Switch ON:  Current flows through coil, stores energy
2. Switch OFF: Coil releases energy through diode
3. Repeat at high frequency (100kHz - 2MHz)

Output voltage controlled by duty cycle:
Vout = Vin × Duty_Cycle
```

### Buck Converter (Step-Down)

```
Vin > Vout (降압)

Example: 12V → 3.3V
Duty Cycle = Vout / Vin = 3.3 / 12 = 27.5%

Efficiency: 80-95% (much better than LDO!)

Popular ICs:
- LM2596: 40V input, 3A output, cheap
- MP1584: 28V input, 3A, tiny
- TPS54331: 28V input, 3A, high efficiency
```

### Boost Converter (Step-Up)

```
Vin < Vout (昇압)

Example: 3.7V (battery) → 5V (USB)

   Vin ──[L]──┬───[Diode]───┬─── Vout
              │             ===
           [Switch]         │
              │             │
   GND ───────┴─────────────┴─── GND

Popular ICs:
- MT3608: 2-24V input, 28V max output, 2A
- TPS61088: 2.3-6V input, 12V output
- XL6009: 5-32V input, 35V max output
```

### Buck-Boost Converter

```
Vin can be > or < Vout

Example: Battery (3.0V - 4.2V) → 3.3V fixed
- When battery full (4.2V): Buck mode
- When battery low (3.0V):  Boost mode

Popular ICs:
- TPS63000: 1.8-5.5V input, 1.8-5.5V output
- LTC3115: 2.7-40V input, 1.8-40V output
```

### SMPS Characteristics

```
✅ Pros:
- High efficiency (80-95%)
- Can step-up or step-down
- Less heat generation
- Good for battery power

❌ Cons:
- More complex (needs inductor, capacitors)
- Switching noise (ripple on output)
- More expensive
- EMI (electromagnetic interference)

Noise mitigation:
- Output capacitors (47µF - 100µF)
- Keep traces short
- Use ground plane
- Add LC filter if needed
```

---

## Battery Basics

### Battery Voltage (Li-ion/LiPo)

```
Single cell Li-ion:
- Fully charged: 4.2V
- Nominal:       3.7V
- Discharged:    3.0V (cutoff)
- Over-discharge: <2.5V (damage!)

Voltage under load drops (internal resistance)
```

### Capacity (Ah, mAh)

```
1000mAh = 1Ah = 1 amp for 1 hour

Example: 3000mAh battery
- 300mA draw → 10 hours
- 1A draw     → 3 hours
- 3A draw     → 1 hour

Real capacity < rated (due to losses, aging)
```

### C-Rate (Charge/Discharge Rate)

```
C-rate = Current / Capacity

Example: 1000mAh battery
- 1C = 1000mA (1 hour discharge)
- 0.5C = 500mA (2 hour discharge)
- 2C = 2000mA (30 min discharge)

Max C-rate depends on battery chemistry:
- Li-ion: 1-2C typical
- LiPo:   5-20C (RC batteries)
- Li-FePO4: 3-5C
```

### Cell Balancing (BMS Basic)

```
Problem: Cells in series drift in voltage
   Cell 1: 4.15V ┐
   Cell 2: 4.05V ├─ Total 12.35V
   Cell 3: 4.15V ┘

If charged to 12.6V:
   Cell 1: 4.3V  ← Overcharged! 🔥
   Cell 2: 4.0V
   Cell 3: 4.3V  ← Overcharged! 🔥

Solution: BMS balances cells
- Passive: Discharge high cells through resistor
- Active:  Transfer energy from high to low cells
```

### Battery Protection (BMS Functions)

```
BMS monitors and protects:
1. Overvoltage (>4.2V per cell)
2. Undervoltage (<3.0V per cell)
3. Overcurrent (>max discharge rate)
4. Short circuit
5. Overtemperature
6. Cell balancing

Result: Disconnect battery if fault detected
```

---

## Motor Control Circuits

### Why Can't GPIO Drive Motor?

```
Motor specs: 12V, 2A (24W)
GPIO specs:  3.3V, 25mA (0.08W)

GPIO current is 80× too small!

Solution: Use power switch (MOSFET or H-bridge)
```

---

## H-Bridge (Motor Direction Control)

### Basic H-Bridge Circuit

```
              +12V (Battery)
               │
         Q1 ┌─┴─┐ Q2
      ──────┤   ├──────
            └─┬─┘
              │
         ┌────┴────┐
         │  Motor  │
         └────┬────┘
              │
         Q3 ┌─┴─┐ Q4
      ──────┤   ├──────
            └─┬─┘
              │
             GND

Q1, Q4 = N-channel or P-channel MOSFETs
```

### Control Logic

```
State 1: Forward (Q1 ON, Q4 ON)
   Current: +12V → Q1 → Motor → Q4 → GND
   Motor spins: Clockwise

State 2: Reverse (Q2 ON, Q3 ON)
   Current: +12V → Q2 → Motor → Q3 → GND
   Motor spins: Counter-clockwise

State 3: Brake (Q3 ON, Q4 ON) or (Q1 ON, Q2 ON)
   Motor terminals shorted → fast stop

State 4: Coast (All OFF)
   Motor freewheels → slow stop

⚠️ NEVER: Q1+Q3 or Q2+Q4 ON simultaneously
   → Short circuit! 🔥 → Shoot-through
```

### Dead-Time Control

```
Problem: MOSFETs don't switch instantly
If Q1 still conducting when Q3 turns on:
   → Momentary short circuit!

Solution: Dead-time delay
   1. Turn OFF Q1
   2. Wait 1-2µs (dead-time)
   3. Turn ON Q3

All H-bridge drivers include dead-time
```

### Popular H-Bridge ICs

```
L298N:  2A per channel, up to 46V
   - Cheap, easy to use
   - Not very efficient (~70%)

DRV8833: 1.5A per channel, up to 10.8V
   - High efficiency (>90%)
   - Integrated current limiting

TB6612FNG: 1.2A per channel, up to 15V
   - Very efficient
   - PWM support

BTS7960: 43A per channel, up to 27V
   - Very high current (for bigger motors)
```

---

## Gate Driver (High-Side Switch)

### The Problem

```
N-channel MOSFET (preferred, better performance)
Needs Gate voltage > Source voltage + Vth

High-side switch:
              +12V
               │
              D│
   Gate ──────┤│── N-CH MOSFET
              S│
               │
           [Load]

When MOSFET on: Source ≈ 12V
Gate needs: >12V + 4V = >16V
But MCU only outputs 3.3V! ❌
```

### Solution 1: P-Channel MOSFET (Simple)

```
              +12V
               │
              S│
   Gate ──────┤│── P-CH MOSFET (upside down!)
              D│
               │
           [Load]

P-channel turns on when Gate < Source
Gate = 0V, Source = 12V → ON ✓
Gate = 12V → OFF

✅ Simple (MCU can drive directly with PNP transistor)
❌ Higher RDS(on) than N-channel
❌ More expensive
```

### Solution 2: Bootstrap Circuit (N-Channel High-Side)

```
Bootstrap capacitor charges when low-side ON
Provides Gate voltage above supply

              +12V ──┬─── VCC
                     │
                   [Bootstrap]
                     │    ┌────┐
                     ├────┤Gate│
                     │    │Drv │
              HS ──[Cap]──┤    │
                     │    └────┘
                    D│
   High-Side ───────┤│
                    S│
                     │
                  [Load]
                     │
                    D│
   Low-Side ────────┤│
                    S│
                     │
                    GND
```

### Gate Driver ICs

```
IR2104:  Bootstrap driver, up to 200V
MCP1407: 6A gate current, fast switching
UCC27211: 4A gate current, 120V

Benefits:
✅ Fast switching (reduces losses)
✅ High gate current
✅ Integrated dead-time
✅ Fault protection
```

---

## Power Dissipation & Thermal Management

### Power Loss in MOSFETs

```
Conduction loss (ON state):
P_cond = I² × RDS(on)

Example: 10A through MOSFET, RDS(on) = 0.01Ω
P_cond = 10² × 0.01 = 1W

Switching loss (during transitions):
Depends on switching frequency and speed

Total loss = P_cond + P_switch
```

### Heatsink Calculation

```
Given:
- Power dissipation: 5W
- Max junction temp: 150°C
- Ambient temp: 50°C
- Junction-to-case: 1°C/W
- Case-to-heatsink: 0.5°C/W

Required heatsink thermal resistance:
R_total = (T_j - T_amb) / P
R_total = (150 - 50) / 5 = 20°C/W

R_heatsink = R_total - R_jc - R_cs
R_heatsink = 20 - 1 - 0.5 = 18.5°C/W

→ Need heatsink rated ≤ 18.5°C/W
```

---

## Practical Examples

### Example 1: Selecting Voltage Regulator

```
Requirements:
- Input: 12V car battery (10-14V range)
- Output: 5V for USB, 500mA
- Efficiency important (battery powered)

Option A: LDO (e.g., LM7805)
   Efficiency: 5/12 = 42%
   Power loss: (12-5) × 0.5 = 3.5W 🔥
   → Not ideal

Option B: Buck SMPS (e.g., LM2596)
   Efficiency: ~85%
   Power loss: 6W × (1-0.85) = 0.9W
   → Much better! ✓
```

### Example 2: Battery Life Calculation

```
System: ESP32 + sensor
- Active: 180mA @ 3.3V
- Sleep:  10µA
- Active 10% of time, sleep 90%

Average current:
I_avg = 0.1 × 180mA + 0.9 × 0.01mA = 18mA

Battery: 2000mAh
Life = 2000mAh / 18mA = 111 hours ≈ 4.6 days
```

### Example 3: Motor Controller Selection

```
Motor: 12V, 5A peak, 2A continuous
RPi GPIO: 3.3V, 16mA max

Can't drive directly! Need H-bridge.

Selection criteria:
- Voltage: ≥12V (with margin, use ≥15V rated)
- Current: ≥5A peak
- PWM frequency: ≥20kHz (for smooth speed control)
- Thermal: Check power dissipation

Good choice: TB6612FNG (1.2A) ❌ Too small
Better:      L298N (2A) ⚠️ Marginal
Best:        BTS7960 (43A) ✓ Plenty of margin
```

---

## Key Takeaways

| Topic | Critical Point |
|-------|---------------|
| **LDO** | Simple, clean, but wastes energy as heat |
| **Buck** | Step-down, efficient, but needs inductor |
| **Boost** | Step-up, for battery to higher voltage |
| **BMS** | Protects battery from over/under voltage/current |
| **H-Bridge** | Controls motor direction, needs dead-time |
| **Gate Driver** | Drives high-side N-channel MOSFETs |

---

## Practical Exercises

1. Calculate efficiency: 9V → 3.3V @ 300mA (LDO vs Buck)
2. Design 3.7V battery → 5V boost converter
3. Calculate battery life for IoT device
4. Select MOSFET for 12V, 10A motor control
5. Calculate heatsink requirement for LDO

---

**Next:** [0.5 Signal Processing](../0.5_Signal_Processing/README.md)
