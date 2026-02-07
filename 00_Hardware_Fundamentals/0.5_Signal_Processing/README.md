# 0.5 Signal Processing

## Overview

Signal processing converts real-world analog signals (temperature, light, sound) into digital data that software can understand. Critical for:
- Sensor interfacing
- Motor control (PWM)
- Audio processing
- Data acquisition

---

## ADC (Analog-to-Digital Converter)

### What ADC Does

```
Analog World          Digital World
(Continuous)          (Discrete)

Temperature ───[ADC]──→ 0x3A5 (value)
0-5V               │    ↓
Infinite values    │    Software
                   │    processes
                   └──→ Display: "25.3°C"
```

---

## ADC Resolution

### Bits and Quantization Levels

```
Resolution determines precision:

8-bit ADC:  2^8  = 256 levels
10-bit ADC: 2^10 = 1024 levels
12-bit ADC: 2^12 = 4096 levels
16-bit ADC: 2^16 = 65536 levels

Higher resolution = finer measurements
```

### Voltage per Step (LSB)

```
LSB = V_ref / (2^n)

Example 1: 10-bit ADC, V_ref = 3.3V
LSB = 3.3V / 1024 = 3.22mV per step

Reading 0x200 (512 decimal):
Voltage = 512 × 3.22mV = 1.65V

Example 2: 12-bit ADC, V_ref = 3.3V
LSB = 3.3V / 4096 = 0.805mV per step
Better precision! ✓
```

### Practical Resolution Example

```
Measuring temperature (0-100°C) with 10mV/°C sensor:
- 0°C   → 0V
- 100°C → 1V

With 10-bit ADC (3.3V ref):
- LSB = 3.22mV
- Temperature resolution = 3.22mV / (10mV/°C) = 0.32°C

With 12-bit ADC:
- LSB = 0.805mV  
- Temperature resolution = 0.081°C (much better!)
```

---

## Sampling Rate (Sample Frequency)

### Nyquist Theorem

```
To accurately capture signal:
f_sample ≥ 2 × f_signal_max

This is the MINIMUM (Nyquist rate)
In practice, use 5-10× for good quality

Example: Audio (20Hz - 20kHz)
Nyquist: 2 × 20kHz = 40kHz minimum
Good:    48kHz or 96kHz
CD quality: 44.1kHz
```

### Aliasing

```
If f_sample < 2 × f_signal:
→ Aliasing (false frequencies appear)

Example: Wheel in movies
- Wheel spins at 30 Hz
- Camera samples at 24 fps
- Appears to spin backwards!

Prevention:
1. Sample fast enough (follow Nyquist)
2. Use anti-aliasing filter (low-pass before ADC)
```

### Typical Sampling Rates

```
Application          Sample Rate
--------------------------------------
Sensor logging       1-100 Hz
Audio (speech)       8 kHz
Audio (music)        44.1 kHz - 192 kHz
Vibration analysis   10-100 kHz
Oscilloscope         100 MHz - 10 GHz
```

---

## Reference Voltage (V_ref)

### Why V_ref Matters

```
ADC measures: 0 to V_ref

Digital value = (V_in / V_ref) × (2^n - 1)

Example: 10-bit ADC, V_in = 1.65V
   If V_ref = 3.3V: value = (1.65/3.3) × 1023 = 512
   If V_ref = 2.5V: value = (1.65/2.5) × 1023 = 676

Changing V_ref changes reading!
```

### V_ref Sources

**Internal Reference**
```
Built into MCU
Typical: 1.2V, 2.5V, or fraction of VDD
✅ Convenient, no external parts
❌ Less accurate (~1-2% error)
❌ Temperature dependent
```

**External Reference**
```
Dedicated reference IC (e.g., TL431, REF3030)
✅ High accuracy (0.05-0.1% error)
✅ Temperature stable
❌ Requires external component
❌ Adds cost

Use for:
- Precision measurements
- Sensor calibration
- Multi-channel matched readings
```

### Maximizing ADC Range

```
Problem: Sensor output 0-1V, ADC range 0-3.3V
   Using only 30% of ADC range → wasting resolution!

Solution 1: Use lower V_ref (1.2V internal)
   Now using 83% of range ✓

Solution 2: Amplify signal
   Op-amp gain of 3× → 0-3V range
   Using 91% of range ✓✓

Solution 3: Both!
   Amplify + optimal V_ref
```

---

## ADC Conversion Time

### How Long ADC Takes

```
Factors:
1. Clock speed (ADC clock)
2. Resolution (more bits = more time)
3. Conversion method (SAR, Sigma-Delta, etc.)

Typical: 1-100 microseconds

Example: STM32F4 ADC
- 12-bit resolution
- 30 MHz ADC clock
- 3 + 12 = 15 cycles
- Time = 15 / 30MHz = 0.5µs
```

### Successive Approximation (SAR) ADC

```
Most common in MCUs

Binary search for voltage:
1. Try V_ref/2:     V_in > V_ref/2? → bit 11 = 1
2. Try V_ref/2 ± V_ref/4: → bit 10 = ?
3. Continue for all bits

Fast (µs), medium accuracy
```

---

## ADC Modes

### Single Conversion

```
Software triggers one conversion

Example:
   ADC_Start();
   while (!ADC_ConversionComplete());
   value = ADC_Read();

Use: Occasional measurements (button voltage)
```

### Continuous Conversion

```
ADC runs continuously

Example:
   ADC_Start_Continuous();
   // ADC keeps converting
   // Read anytime: value = ADC_Read();

Use: Real-time monitoring
```

### Scan Mode (Multiple Channels)

```
ADC scans through multiple inputs automatically

Channels: 0 → 1 → 2 → 3 → back to 0

Example: Monitor 4 sensors
   ADC_Scan([CH0, CH1, CH2, CH3]);
   // Results stored in array
```

### DMA Integration

```
ADC → DMA → Memory (no CPU involvement!)

   [ADC] ──→ [DMA Controller] ──→ [Buffer in RAM]
             (automatic)

✅ CPU free for other tasks
✅ No missed samples
✅ Best for high-speed continuous acquisition

Example code:
   uint16_t adc_buffer[100];
   ADC_DMA_Start(adc_buffer, 100);
   // ADC fills buffer via DMA
   // CPU notified when buffer full
```

---

## PWM (Pulse Width Modulation)

### What is PWM?

```
Digital signal with varying duty cycle
Simulates analog voltage through averaging

   High ───┐     ┐     ┐
           │     │     │
   Low  ───┘─────┘─────┘───
       
       ←─ Period ─→
       ←T_on→
       
Duty Cycle = (T_on / Period) × 100%
```

---

## Duty Cycle

### Calculating Duty Cycle

```
Duty Cycle = (T_on / T_period) × 100%

Example 1: 1ms on, 4ms off
   Period = 5ms
   Duty = 1/5 = 20%

Example 2: 3ms on, 2ms off
   Period = 5ms
   Duty = 3/5 = 60%

Average voltage = VDD × Duty_Cycle
   20% duty @ 5V → 1V average
   60% duty @ 5V → 3V average
```

### Typical Applications

```
Application        Duty Cycle
-------------------------------
LED brightness:    0-100% (dimming)
Motor speed:       0-100% (speed control)
Servo position:    5-10% (1-2ms pulse)
Buzzer tone:       50% (square wave)
DAC simulation:    0-100% (voltage control)
```

---

## PWM Frequency

### Frequency Selection

```
Frequency = 1 / Period

Low frequency (100 Hz):
   ✅ Easy to generate
   ❌ Visible flicker (LED)
   ❌ Audible noise (motor)

Medium frequency (1-20 kHz):
   ✅ No flicker
   ✅ Silent operation
   ✅ Good for most applications

High frequency (>50 kHz):
   ✅ Very smooth
   ❌ Switching losses
   ❌ EMI issues
```

### Application-Specific Frequencies

```
LED dimming:      1-10 kHz (avoid flicker)
Motor control:    20-40 kHz (silent, efficient)
Servo control:    50 Hz (20ms period, standard)
SMPS:             100 kHz - 2 MHz (high efficiency)
Audio:            31.25 kHz - 96 kHz (PWM audio)
```

---

## PWM Configuration

### Timer Setup Example

```c
// Generate 1kHz PWM, 50% duty on STM32
// Assumptions: Timer clock = 84 MHz

// Period for 1kHz: T = 1ms = 1000µs
// Timer counts needed: 84 MHz / 1 kHz = 84000

TIM2->PSC = 0;           // Prescaler = 1
TIM2->ARR = 84000 - 1;   // Auto-reload (period)
TIM2->CCR1 = 42000;      // Compare value (50% duty)

// Duty cycle = CCR1 / ARR = 42000 / 84000 = 50%

// To change duty cycle:
TIM2->CCR1 = 25200;      // 30% duty
TIM2->CCR1 = 67200;      // 80% duty
```

### Variable Duty Cycle

```c
// Set brightness (0-100%)
void set_led_brightness(uint8_t percent) {
    if (percent > 100) percent = 100;
    
    uint32_t ccr = (TIM2->ARR + 1) * percent / 100;
    TIM2->CCR1 = ccr;
}

// Usage:
set_led_brightness(0);    // Off
set_led_brightness(50);   // Half brightness
set_led_brightness(100);  // Full brightness
```

---

## PWM for Motor Control

### Speed Control

```
DC Motor speed ∝ Average voltage ∝ Duty cycle

0% duty   → Motor stopped
50% duty  → Half speed
100% duty → Full speed

With H-bridge: Also control direction
   Forward:  PWM on Q1, Q4 ON
   Reverse:  PWM on Q2, Q3 ON
   Brake:    Both low-side ON
```

### PWM Frequency Considerations

```
Too low (<1 kHz):
   ❌ Audible whine
   ❌ Jerky motion
   ❌ More torque ripple

Good (20-40 kHz):
   ✅ Silent
   ✅ Smooth motion
   ✅ Reasonable efficiency

Too high (>100 kHz):
   ❌ Excessive switching losses
   ❌ EMI issues
   ❌ Gate drive problems
```

---

## DAC (Digital-to-Analog Converter)

### What DAC Does

```
Opposite of ADC:
Digital value ──[DAC]──→ Analog voltage

Example: Audio playback
   MP3 data → DAC → Speaker amplifier → Sound
```

### DAC Resolution

```
Same concept as ADC:

8-bit DAC:  256 voltage levels
12-bit DAC: 4096 voltage levels

Example: 8-bit DAC, V_ref = 3.3V
   Step size = 3.3V / 256 = 12.9mV
   
   Write 0x80 (128) → 1.65V output
   Write 0xFF (255) → 3.3V output
```

### DAC Applications

```
Audio output:      Music, speech synthesis
Function generator: Sine, triangle, sawtooth waves
Analog control:    Variable power supply
Sensor simulation: Test equipment
Offset/bias:       ADC reference adjustment
```

---

## Filtering

### Why Filtering?

```
Real-world signals have noise:
   Desired signal + Noise → ADC → ???

Filtering removes unwanted frequencies
   Low-pass:  Remove high-frequency noise
   High-pass: Remove DC offset, low-frequency drift
   Band-pass: Keep only frequency range of interest
```

---

## RC Low-Pass Filter

### Passive RC Filter

```
    Signal_in ───[R]───┬──── Signal_out
                       │
                      ═══ C
                       │
                      GND

Cutoff frequency:
f_c = 1 / (2π × R × C)

Above f_c: Signal attenuated (reduced)
Below f_c: Signal passes through
```

### Example: Anti-Aliasing Filter

```
Sensor output (0-100 Hz) with noise (1 kHz+)
ADC sampling at 1 kHz

Design low-pass filter: f_c = 200 Hz
   R = 10kΩ
   C = 1 / (2π × 10kΩ × 200Hz) = 80nF
   → Use 82nF (standard value)

Result: Removes high-frequency noise before ADC
```

---

## LC Filter

### Inductor-Capacitor Filter

```
Used for power supplies (switching regulators)

         L
    ──∿∿∿──┬────
           │
          ═══ C
           │
          GND

Better filtering than RC:
- Steeper rolloff
- Lower loss
- More expensive (inductor)
```

---

## Practical Examples

### Example 1: Temperature Sensor

```
Problem: Read TMP36 temperature sensor
   Output: 10mV/°C, 0.5V at 0°C
   Range: 0°C to 100°C → 0.5V to 1.5V

Solution:
1. ADC: 12-bit, V_ref = 3.3V
   LSB = 3.3V / 4096 = 0.805mV

2. Read ADC value
   voltage = adc_value × 0.805mV

3. Calculate temperature
   temp_C = (voltage - 500mV) / 10mV

Code:
   uint16_t adc = ADC_Read();
   float voltage = adc * 0.000805;  // in V
   float temp = (voltage - 0.5) / 0.01;
   printf("Temperature: %.1f C\n", temp);
```

### Example 2: LED Brightness Control

```
Problem: Fade LED in and out smoothly

Solution: PWM with changing duty cycle

Code:
   for (int brightness = 0; brightness <= 100; brightness++) {
       set_led_brightness(brightness);
       delay_ms(20);  // 20ms per step
   }
   // Total fade-in: 2 seconds

For smoother:
   - Use smaller steps (0.1% instead of 1%)
   - Shorter delays
   - Exponential curve (human eye perceives logarithmically)
```

### Example 3: Motor Speed Control

```
Problem: Control DC motor 0-100% speed with knob

Solution:
1. Read potentiometer with ADC (0-4095)
2. Map to PWM duty cycle (0-100%)
3. Apply PWM to motor H-bridge

Code:
   uint16_t pot = ADC_Read();           // 0-4095
   uint8_t speed = pot * 100 / 4095;    // 0-100%
   
   set_motor_speed(speed);
```

---

## Key Takeaways

| Concept | Critical Point |
|---------|---------------|
| **ADC Resolution** | More bits = finer measurements |
| **Sampling Rate** | Must be ≥ 2× signal frequency (Nyquist) |
| **V_ref** | Defines full-scale voltage, affects accuracy |
| **PWM Duty Cycle** | Controls average voltage/power |
| **PWM Frequency** | 20kHz typical for motors (silent) |
| **Filtering** | Remove noise before ADC sampling |

---

## Practical Exercises

1. Calculate LSB for 10-bit ADC with 5V reference
2. Determine sampling rate for 5kHz signal (with margin)
3. Design RC filter with 1kHz cutoff
4. Calculate PWM settings for 25kHz, 30% duty
5. Convert ADC reading to temperature (with sensor datasheet)
6. Implement software low-pass filter (moving average)

---

## Common Mistakes

❌ **Sampling too slow** → Aliasing, missing signal
❌ **Wrong V_ref** → All readings scaled incorrectly  
❌ **No input filtering** → Noisy ADC readings
❌ **PWM too slow** → Audible noise, flicker
❌ **Forgetting stabilization time** → First ADC reading wrong

---

**Congratulations!** You've completed all Hardware Fundamentals sections.

**Next:** [01 Embedded Fundamentals](../../01_Embedded_Fundamentals/README.md)
