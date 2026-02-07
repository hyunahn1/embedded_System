# 00. Hardware Fundamentals (Layer 0)

## Overview

This section covers the essential hardware knowledge that every embedded software engineer should know. Understanding these fundamentals is crucial for:
- Reading and interpreting circuit schematics
- Debugging hardware-related issues
- Communicating effectively with hardware engineers
- Making informed design decisions

## Why Hardware Knowledge Matters for SW Engineers

**Real-World Scenarios:**
- 🐛 "Why isn't my GPIO reading the sensor?" → Missing pull-up resistor
- 🔥 "The MCU keeps resetting randomly" → Inadequate decoupling capacitor
- ⚡ "CAN bus communication fails intermittently" → Improper termination resistor
- 🚗 "Motor control is unstable" → Gate driver circuit issue

**You don't need to design circuits, but you MUST:**
- Read datasheets and schematics
- Understand why hardware is designed a certain way
- Debug at the hardware-software interface
- Collaborate with hardware team

## Contents

### 0.1 Circuit Theory
- Basic electrical laws (Ohm's Law, Kirchhoff's Laws)
- Voltage, current, power calculations
- Ground concepts and proper grounding
- Short circuit vs open circuit

### 0.2 Electronic Components
- Resistors (pull-up, pull-down, voltage divider)
- Capacitors (decoupling, filtering, timing)
- Inductors and transformers
- Diodes (protection, rectification)
- Transistors and MOSFETs (switching, amplification)

### 0.3 Digital Logic & Interface
- Logic gates and Boolean algebra
- Logic levels (TTL vs CMOS)
- GPIO output modes (push-pull, open-drain)
- Level shifters and signal conditioning

### 0.4 Power Electronics
- Voltage regulators (LDO, buck, boost)
- Battery basics and BMS concepts
- Motor control circuits (H-bridge, gate driver)
- Power dissipation and thermal management

### 0.5 Signal Processing
- ADC (sampling, resolution, reference voltage)
- PWM (duty cycle, frequency)
- DAC and analog output
- Filtering (RC, LC filters)

### 0.6 Schematic Reading & Tools
- Reading circuit schematics
- Component symbols and conventions
- Datasheet interpretation
- Using multimeter, oscilloscope, logic analyzer

---

## Learning Path

**For Complete Beginners:**
1. Start with 0.1 (Circuit Theory) - understand the basic laws
2. Move to 0.2 (Components) - learn what each part does
3. Study 0.3 (Digital Logic) - bridge to software
4. Read 0.6 (Schematics) alongside everything
5. Practice with 0.4 and 0.5 as needed

**For Software Engineers with Some EE Background:**
1. Skim 0.1 and 0.2 for review
2. Focus on 0.3 (Digital Logic) - critical for embedded
3. Study 0.6 (Schematic Reading) thoroughly
4. Reference 0.4 and 0.5 when working with specific systems

---

## Prerequisites

- Basic physics knowledge (electricity concepts)
- Algebra (for calculations)
- Curiosity about how hardware works

No prior electronics experience required, but willingness to learn is essential.

---

## Practical Application

After completing this section, you should be able to:
- ✅ Read a typical MCU circuit schematic
- ✅ Understand GPIO circuit configurations
- ✅ Identify common protection circuits
- ✅ Debug basic hardware issues
- ✅ Interpret datasheet electrical specifications
- ✅ Use measurement tools effectively
- ✅ Communicate with hardware engineers

---

## How Much Hardware Knowledge Do You Need?

**Minimum (Bare Minimum for SW Dev):**
- Ohm's Law, basic voltage/current
- Pull-up/pull-down resistors
- Decoupling capacitors
- Logic levels (3.3V vs 5V)
- Reading GPIO schematics

**Recommended (Professional Embedded Dev):**
- Everything in this Layer 0 section
- Common circuit patterns
- Datasheet analysis
- Oscilloscope usage

**Advanced (System-Level or EV/Motor Control):**
- Layer 0 + power electronics
- Motor control theory
- EMI/EMC considerations
- High-speed signal integrity

---

## Resources

- Component datasheets (always start here)
- "The Art of Electronics" by Horowitz & Hill
- "Practical Electronics for Inventors" by Scherz & Monk
- EEVblog (YouTube) for practical circuits
- Falstad Circuit Simulator (online simulator)

---

## Important Notes

⚠️ **Safety First:**
- Always check voltage levels before connecting
- Use current limiting when testing
- Don't hot-plug connectors on powered circuits
- ESD precautions with sensitive components

💡 **Learning Tips:**
- Simulate circuits before building (Falstad, LTSpice)
- Use a multimeter to verify your understanding
- Keep a personal library of circuit patterns
- Ask "why is this component here?" when reading schematics

---

**Next Step:** Start with [0.1 Circuit Theory](0.1_Circuit_Theory/README.md)
