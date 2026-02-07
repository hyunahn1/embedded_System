# 0.1 Circuit Theory

## Overview

Understanding basic circuit theory is fundamental for embedded software engineers. These principles explain **why circuits behave the way they do** and help you debug hardware-software interface issues.

---

## Ohm's Law (옴의 법칙)

### The Most Important Equation in Electronics

```
V = I × R

V = Voltage (전압, Volts)
I = Current (전류, Amperes)
R = Resistance (저항, Ohms, Ω)
```

### Practical Examples

#### Example 1: LED Current Limiting
```
Problem: Power an LED (2V forward voltage) from 5V supply
LED requires 20mA for proper brightness

Solution: Calculate resistor needed
V_resistor = V_supply - V_led = 5V - 2V = 3V
R = V / I = 3V / 0.02A = 150Ω

Choose standard value: 150Ω or 220Ω (safer)
```

#### Example 2: Pull-up Resistor Selection
```
Problem: Choose pull-up resistor for I2C bus
- Supply voltage: 3.3V
- Desired current when pulled low: ~1mA
- Bus capacitance: 100pF

R = V / I = 3.3V / 0.001A = 3.3kΩ

Typical values: 2.2kΩ - 10kΩ
Faster bus → smaller resistor
Lower power → larger resistor
```

---

## Power (전력)

### Power Dissipation

```
P = V × I  (Watts)
P = I² × R
P = V² / R
```

### Why This Matters

**Component Selection:**
- Resistor must handle power dissipation
- If P > resistor rating → **component burns out!** 🔥

#### Example: Voltage Regulator Heat
```
LDO regulator:
V_in = 12V
V_out = 5V
I_out = 500mA

Power dissipated as heat:
P = (V_in - V_out) × I_out
P = (12V - 5V) × 0.5A = 3.5W

→ Needs heatsink!
```

---

## Kirchhoff's Laws

### Kirchhoff's Current Law (KCL)

**"Current going in = Current coming out"**

```
At any node: ΣI_in = ΣI_out
```

**Example: Current Distribution**
```
       I_total = 100mA
           |
       ┌───┴───┐
       ↓       ↓
     I1=30mA  I2=70mA
```

**Application:** Understanding current flow in circuits, debugging short circuits

### Kirchhoff's Voltage Law (KVL)

**"Sum of voltages around a closed loop = 0"**

```
In any closed loop: ΣV = 0
```

**Example: Voltage Divider**
```
    5V
     |
    [R1=10kΩ]  ← V1
     |
     +-------- V_out (2.5V)
     |
    [R2=10kΩ]  ← V2
     |
    GND

V_supply = V1 + V2
5V = V1 + V2

If R1=R2, then V1=V2=2.5V
```

---

## Voltage Divider (전압 분배)

### Formula

```
        V_in
         |
        [R1]
         |
         +-----> V_out
         |
        [R2]
         |
        GND

V_out = V_in × (R2 / (R1 + R2))
```

### Applications

**1. Level Shifting (ADC Input)**
```
Problem: 5V sensor output, 3.3V ADC input

Solution:
R1 = 1kΩ
R2 = 2kΩ
V_out = 5V × (2kΩ / 3kΩ) = 3.33V ✓
```

**2. Resistive Sensor Reading**
```
Temperature sensor (NTC thermistor):
- Varies resistance with temperature
- Use voltage divider to convert to voltage
- Read with ADC
```

⚠️ **Warning:** Voltage dividers draw continuous current → power waste
- Only use for sensing, not for powering devices

---

## Ground (접지)

### Types of Ground

**1. Common Ground (Signal Ground)**
```
All signals reference this point (0V)
```

**2. Chassis Ground**
```
Metal enclosure, safety ground
May be connected to signal ground at ONE point only
```

**3. Earth Ground**
```
Literally connected to earth
For safety (AC power systems)
```

### Ground Loops and Star Grounding

**Problem: Ground Loop**
```
    MCU ─────[long wire]───── Sensor
     |                          |
     GND1 ─── [ground plane] ─── GND2
     
If GND1 ≠ GND2 (voltage difference) → noise!
```

**Solution: Star Grounding**
```
              Power Supply GND (star center)
                    |
        ┌──────┬────┴────┬──────┐
        |      |         |      |
       MCU  Sensor   Motor   ADC
```

---

## Impedance (임피던스)

### DC Resistance vs AC Impedance

**Resistance (R):** Opposes DC current
**Impedance (Z):** Opposes AC current (includes capacitive and inductive effects)

```
Z = R + jX

where X = reactance (capacitive or inductive)
```

### Why This Matters

**High-Speed Digital Signals:**
- Traces on PCB have impedance (typically 50Ω or 75Ω)
- Impedance mismatch → signal reflections → data corruption

**Example: USB Differential Pair**
- Requires 90Ω differential impedance
- PCB designer controls trace width/spacing

---

## Short Circuit vs Open Circuit

### Short Circuit (단락, 쇼트)

**Unintended low-resistance path**

```
    Battery (+)
        |
        X ← SHORT (0Ω wire)
        |
    Battery (-)
    
Result: I = V/R = V/0 = ∞
→ Fuse blows, component burns! 🔥
```

**Common Causes:**
- Solder bridge between pins
- Wire insulation damaged
- Liquid spill on PCB

### Open Circuit (개방, 끊김)

**Unintended broken connection**

```
    MCU GPIO ─────╳───── LED
                  ^
               open (broken wire)
    
Result: I = 0, LED doesn't light
```

**Common Causes:**
- Cold solder joint
- Broken wire
- Loose connector

---

## Practical Debugging Examples

### Example 1: GPIO Not Reading Correctly

**Symptom:** GPIO input reads random values (fluctuates between 0 and 1)

**Diagnosis:**
```
    3.3V
     |
     ?  ← Missing pull-up!
     |
    GPIO (floating)
```

**Solution:** Add pull-up or pull-down resistor
```
    3.3V
     |
    [10kΩ]  ← Pull-up resistor
     |
    GPIO ────[Switch]──── GND
```

### Example 2: MCU Resets Randomly

**Symptom:** System resets when motor starts

**Diagnosis:**
- Motor draws large current
- Voltage drops below MCU minimum
- Brown-out reset triggered

**Solution:**
1. Add decoupling capacitors near MCU power pins
2. Separate power supply for motor
3. Use voltage regulator with adequate current rating

---

## Key Concepts Summary

| Concept | Formula | Application |
|---------|---------|-------------|
| Ohm's Law | V = I × R | Calculate resistor values |
| Power | P = V × I | Component heat dissipation |
| Voltage Divider | V_out = V_in × R2/(R1+R2) | Level shifting, sensing |
| KCL | ΣI_in = ΣI_out | Current distribution |
| KVL | ΣV = 0 | Voltage analysis |

---

## Practical Exercises

1. **Calculate LED resistor:** 
   - Supply: 3.3V, LED forward voltage: 2.1V, LED current: 15mA
   
2. **Design voltage divider:**
   - Convert 5V to 1.8V for ADC reference
   
3. **Power calculation:**
   - 7805 regulator, input 12V, output 5V @ 200mA. How much heat?
   
4. **Debug floating input:**
   - GPIO reads 0.8V when button not pressed. What's wrong?

---

## Tools for Practice

- **Falstad Circuit Simulator** (online, free)
- **LTSpice** (professional, free)
- **Multimeter** (measure real circuits)

---

## References

- "The Art of Electronics" by Horowitz & Hill (Chapter 1)
- "Practical Electronics for Inventors" (Chapter 2)
- EEVblog tutorials on YouTube

---

**Next:** [0.2 Electronic Components](../0.2_Electronic_Components/README.md)
