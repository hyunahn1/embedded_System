# 0.3 Digital Logic & Interface

## Overview

Digital logic is where hardware meets software. Understanding these concepts is critical for:
- GPIO configuration (push-pull vs open-drain)
- Communication protocols (I2C, SPI, UART, CAN)
- Interfacing between different voltage levels
- Debugging signal integrity issues

---

## Logic Gates & Boolean Algebra

### Basic Logic Gates

```
AND:  Output = 1 only if ALL inputs are 1
      A B | OUT
      0 0 |  0
      0 1 |  0
      1 0 |  0
      1 1 |  1

OR:   Output = 1 if ANY input is 1
      A B | OUT
      0 0 |  0
      0 1 |  1
      1 0 |  1
      1 1 |  1

NOT:  Output = opposite of input
      A | OUT
      0 |  1
      1 |  0

XOR:  Output = 1 if inputs are DIFFERENT
      A B | OUT
      0 0 |  0
      0 1 |  1
      1 0 |  1
      1 1 |  0

NAND: NOT + AND (universal gate)
NOR:  NOT + OR (universal gate)
```

### Software Connection (Bit Operations)

```c
// C bit operations mirror logic gates
uint8_t a = 0b10101010;
uint8_t b = 0b11001100;

uint8_t result_and = a & b;   // AND
uint8_t result_or  = a | b;   // OR
uint8_t result_xor = a ^ b;   // XOR
uint8_t result_not = ~a;      // NOT

// Set bit
register |= (1 << bit);        // OR operation

// Clear bit
register &= ~(1 << bit);       // AND with inverted mask

// Toggle bit
register ^= (1 << bit);        // XOR operation

// Test bit
if (register & (1 << bit)) {   // AND operation
    // bit is set
}
```

---

## Logic Levels (TTL vs CMOS)

### TTL (Transistor-Transistor Logic) - 5V

```
VCC = 5V

VIH (Input High): ≥ 2.0V
VIL (Input Low):  ≤ 0.8V
VOH (Output High): ≥ 2.4V
VOL (Output Low):  ≤ 0.4V

Noise Margin:
- High: VOH - VIH = 2.4V - 2.0V = 0.4V
- Low:  VIL - VOL = 0.8V - 0.4V = 0.4V
```

### CMOS (Complementary Metal-Oxide-Semiconductor) - 3.3V

```
VDD = 3.3V

VIH (Input High): ≥ 2.0V (or 0.7 × VDD = 2.31V)
VIL (Input Low):  ≤ 0.8V (or 0.3 × VDD = 0.99V)
VOH (Output High): ≥ 2.4V (or 0.8 × VDD = 2.64V)
VOL (Output Low):  ≤ 0.4V (or 0.2 × VDD = 0.66V)

Advantages:
✅ Lower power consumption
✅ Higher noise immunity
✅ Wider voltage range (1.8V - 5V depending on family)
```

### Voltage Level Compatibility

```
Case 1: 5V TTL → 3.3V CMOS Input
   5V output → 3.3V input
   ⚠️ DANGER! Can damage 3.3V IC
   → Need level shifter or voltage divider

Case 2: 3.3V CMOS → 5V TTL Input
   3.3V output → 5V input
   3.3V ≥ 2.0V (VIH for TTL) ✓
   → Usually works, but check datasheet

Case 3: 3.3V CMOS → 3.3V CMOS
   ✓ Direct connection

Case 4: 5V TTL → 5V TTL
   ✓ Direct connection
```

---

## Level Shifters (Voltage Translation)

### Why Level Shifters?

```
Problem: 3.3V MCU ↔ 5V Sensor

Direct connection → 5V damages 3.3V MCU! 🔥
```

### Method 1: Resistor Divider (Output Only)

```
5V Sensor Output → 3.3V MCU Input

    5V Sensor
       |
      [R1] 2kΩ
       |
       +──── To 3.3V MCU input (3.33V)
       |
      [R2] 1kΩ
       |
      GND

V_out = 5V × (1kΩ / 3kΩ) = 1.67V... too low!

Better ratio:
R1 = 1kΩ, R2 = 2kΩ → V_out = 3.33V ✓

⚠️ Limitations:
- One direction only (can't drive 5V)
- Output impedance affects speed
- Wastes power
```

### Method 2: MOSFET Level Shifter (Bidirectional)

```
3.3V Domain              5V Domain
    |                       |
   [10kΩ]                 [10kΩ]
    |                       |
    +───[MOSFET]───────────+
    |   G──S     D          |
   3.3V Line            5V Line

Works for I2C (bidirectional, open-drain)
```

### Method 3: Dedicated Level Shifter IC (Best)

```
Popular ICs:
- TXS0108E (8-channel, bidirectional, auto-direction)
- TXB0108 (8-channel, bidirectional)
- 74LVC245 (8-channel, unidirectional with direction pin)

Example: TXS0108E
   VCCA (3.3V)  ┌──────┐  VCCB (5V)
      A1 ───────┤      ├─────── B1
      A2 ───────┤      ├─────── B2
      ...       │ IC   │       ...
     GND ───────┤      ├─────── GND
                └──────┘

✅ Fast (up to 100MHz+)
✅ Bidirectional
✅ Auto-direction sensing
✅ Low power
```

---

## GPIO Output Modes

### Push-Pull Output (Standard)

```
VDD ────┐
        │
    ┌───┴───┐
    │  High │ P-MOSFET (pulls to VDD)
    │  Side │
    └───┬───┘
        │
        ├───── Output Pin
        │
    ┌───┴───┐
    │  Low  │ N-MOSFET (pulls to GND)
    │  Side │
    └───┬───┘
        │
       GND

States:
- High: P-MOSFET ON, N-MOSFET OFF → Output = VDD
- Low:  P-MOSFET OFF, N-MOSFET ON → Output = GND

✅ Can drive both HIGH and LOW actively
✅ Fast switching
✅ Most common for LEDs, simple outputs

❌ Cannot be used for multi-master buses (I2C)
❌ Multiple outputs driving same line → short circuit!
```

### Open-Drain Output (★ I2C, Multi-Master)

```
VDD ────[Pull-up R]──┬──── Output Pin
                     │
                 ┌───┴───┐
                 │  Low  │ N-MOSFET only
                 │  Side │
                 └───┬───┘
                     │
                    GND

States:
- Low:  N-MOSFET ON  → Output = GND (actively pulled)
- High: N-MOSFET OFF → Output = VDD (through pull-up, weak)

✅ Multiple devices can drive same line (wired-AND)
✅ Level shifting possible (different pull-up voltages)
✅ Required for I2C

❌ Slower switching (weak pull-up)
❌ Requires external pull-up resistor
```

### Open-Drain vs Push-Pull Comparison

```
Feature          | Push-Pull      | Open-Drain
-----------------|----------------|------------------
Drive HIGH       | Active (strong)| Passive (pull-up)
Drive LOW        | Active (strong)| Active (strong)
Multi-master     | ❌ NO          | ✅ YES
Speed            | Fast           | Slower
External R       | Optional       | Required
Use cases        | LED, simple IO | I2C, multi-drop
```

### When to Use Each

**Push-Pull:**
- Driving LEDs
- Simple digital outputs
- SPI (MOSI, MISO, SCK)
- UART (TX, RX)

**Open-Drain:**
- I2C (SDA, SCL) - **REQUIRED!**
- Interrupt lines (multiple sources)
- Reset lines (multiple sources can pull low)
- Wired-OR/AND logic

---

## High-Impedance (Hi-Z) State

### What is Hi-Z?

```
"High Impedance" = "Disconnected" = "Floating"

Both transistors OFF:
   P-MOSFET: OFF
   N-MOSFET: OFF
   
   Output pin is effectively disconnected
   (very high resistance, >10MΩ)
```

### When Used

**Input Mode:**
- GPIO configured as input is Hi-Z
- Allows external device to drive the pin
- Needs pull-up/pull-down to prevent floating

**Tri-State Buffers:**
- Can be enabled/disabled
- Used in bus systems (multiple devices sharing lines)

```
Example: SPI with multiple slaves

Master ──SPI───┬───[Slave 1] (Hi-Z when not selected)
               ├───[Slave 2] (Hi-Z when not selected)
               └───[Slave 3] (Active when selected)
```

---

## Communication Physical Layers

### Single-Ended Signaling

```
One signal wire + ground reference

Example: UART, SPI
    TX ──────[signal]─────── RX
   GND ──────[ground]─────── GND

✅ Simple, cheap
❌ Susceptible to noise (common-mode noise)
❌ Ground potential difference issues
```

### Differential Signaling (★ CAN, RS-485)

```
Two wires: Signal+ and Signal-
Data encoded as voltage DIFFERENCE

Example: CAN Bus
    CAN_H ───────────────────
    CAN_L ───────────────────

Dominant (0): CAN_H > CAN_L (by ~2V)
Recessive (1): CAN_H ≈ CAN_L

Receiver looks at: V_diff = V(CAN_H) - V(CAN_L)
```

### Why Differential is Better

```
Problem: Noise affects both wires equally

   Original:     +2V (H-L)
                 
   Noise added:  +2V + noise (both lines)
                 
   Difference:   Still +2V! (noise cancels)

✅ Excellent noise immunity
✅ No ground reference needed
✅ Longer distances (CAN: 40m @ 1Mbps)
✅ Higher speeds

Used in:
- CAN bus (automotive)
- RS-485 (industrial)
- USB (differential pair D+/D-)
- Ethernet (twisted pair)
- LVDS (high-speed data)
```

### RS-485 Characteristics

```
Differential signaling: A and B lines

Logic 1: A > B + 200mV
Logic 0: A < B - 200mV

✅ Multi-drop (up to 32 nodes, 256 with repeaters)
✅ Long distance (1200m @ 100kbps)
✅ Half-duplex or full-duplex
✅ Noise immunity

Termination: 120Ω resistor at BOTH ends
```

### CAN Bus Characteristics

```
Two wires: CAN_H and CAN_L

Dominant (0):
   CAN_H = ~3.5V
   CAN_L = ~1.5V
   Difference = 2V

Recessive (1):
   CAN_H = 2.5V
   CAN_L = 2.5V
   Difference = 0V

✅ Multi-master (arbitration by ID)
✅ Error detection (CRC, ACK)
✅ Fault confinement (bad nodes isolate)

Termination: 120Ω resistor at BOTH ends
```

---

## Signal Integrity Basics

### Rise Time and Fall Time

```
       VDD ──────┐
                 │╱  Rise time (10% to 90%)
                 │
       0V ───────┘

Fast rise time: Sharp edges, high-frequency content
Slow rise time: Rounded edges, more susceptible to noise

Typical values:
- GPIO toggle: 1-10 ns
- I2C: 100-300 ns (intentionally slowed for stability)
```

### Pull-up/Pull-down Effect on Speed

```
Smaller resistor → Faster rise time → More power
Larger resistor  → Slower rise time → Less power

I2C example:
- 400kHz (Fast mode): 2.2kΩ pull-up
- 100kHz (Standard):  4.7kΩ pull-up
- 10kHz (Low power):  10kΩ pull-up
```

### Termination Resistors

```
Why needed for long wires:
- Signal reflects at end of line
- Causes ringing, double-edges
- Termination resistor absorbs reflection

Termination value = Characteristic impedance
- CAN: 120Ω
- RS-485: 120Ω
- Ethernet: 100Ω
```

---

## Practical Examples

### Example 1: I2C Configuration

```
Problem: I2C bus not working

Check:
1. Open-drain output configured? ✓
2. Pull-up resistors present? ✓
3. Correct voltage levels? ✓
4. SDA and SCL not swapped? ✓

Typical I2C configuration:
   +3.3V
     |
    [4.7kΩ]  [4.7kΩ]
     |        |
    SDA ─────●───── (to all devices)
    SCL ─────●───── (to all devices)
```

### Example 2: 3.3V MCU controlling 12V relay

```
Cannot drive relay directly from GPIO!
- GPIO max: 25mA @ 3.3V
- Relay coil: 50mA @ 12V

Solution: MOSFET switch
   +12V
     |
   [Relay Coil]
     |
    D|
  ──|─┤ N-Channel MOSFET
  | S|
  |  |
 [10kΩ] (pull-down)
  |  |
 GPIO GND
```

### Example 3: Bidirectional Level Shifting for I2C

```
3.3V MCU ↔ 5V sensor on I2C bus

Solution: MOSFET bidirectional shifter
    3.3V          5V
     |            |
    [4.7kΩ]     [4.7kΩ]
     |            |
     ├─[MOSFET]──┤
    SDA_3V3    SDA_5V
```

---

## Key Takeaways

| Concept | Critical Point |
|---------|---------------|
| **Logic Levels** | Always check voltage compatibility! |
| **Push-Pull** | Standard output, cannot multi-drop |
| **Open-Drain** | Required for I2C, needs pull-up |
| **Level Shifters** | Use for 3.3V ↔ 5V interfaces |
| **Differential** | Best noise immunity (CAN, RS-485) |

---

## Practical Exercises

1. Calculate I2C pull-up resistor for 400kHz bus with 100pF capacitance
2. Design level shifter for 3.3V → 5V UART
3. Explain why I2C requires open-drain (not push-pull)
4. Debug: GPIO configured as push-pull, I2C not working
5. Calculate termination resistor for 10m CAN cable

---

## Common Mistakes

❌ **Using push-pull for I2C** → Bus conflicts, crashes
❌ **Forgetting pull-ups on open-drain** → Floating lines
❌ **Direct 5V to 3.3V connection** → Damaged IC
❌ **Missing CAN termination** → Unreliable communication
❌ **Mixing logic levels without shifter** → Unpredictable behavior

---

**Next:** [0.4 Power Electronics](../0.4_Power_Electronics/README.md)
