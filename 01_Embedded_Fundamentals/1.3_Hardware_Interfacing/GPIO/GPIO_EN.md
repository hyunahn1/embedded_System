# GPIO

## GPIO Concept

### Easy Understanding

**GPIO = Universal Socket**

Just like an electrical outlet can be used for various purposes, GPIO pins can be used in multiple ways:
- Can be used as output to turn on LEDs
- Can be used as input to read buttons
- Can also be used for communication (UART, I2C)

**Key Point:** The role of the pin is determined by the program (code)!

```c
void setup() {
    pinMode(13, OUTPUT);     // Set pin 13 as output
    pinMode(12, INPUT);      // Set pin 12 as input
    digitalWrite(13, HIGH);  // Turn on pin 13 (3.3V output)
    digitalWrite(13, LOW);   // Turn off pin 13 (0V output)
}
```

### Technical Definition

GPIO stands for **General-Purpose Input/Output**, meaning **general-purpose input/output pins**.

The term "general-purpose" means it can be used for various purposes. In other words, whether a pin is used as input or output is determined by the program running on the microcontroller.

For example, when a pin is set as input, whether that input pin is connected to the microcontroller's positive power voltage through an internal pull-up resistor is also determined by the program.

### Using as Digital Output

GPIO pins are often used as digital outputs, meaning they are used to turn external devices or components ON/OFF.

GPIO pins can directly connect components like LEDs to turn them ON/OFF, or connect photocouplers or relays to control the supply or interruption of high voltage or current to devices.

**Voltage Levels:**
- **HIGH output:** Microcontroller's power voltage (5V or 3.3V) appears on the pin
- **LOW output:** 0V appears on the pin

**Compatibility Note:**
- 5V microcontrollers mostly work with 3.3V voltage, but the reverse may not work.

### Using as Digital Input

To make the microcontroller respond to incoming inputs and perform actions, set GPIO as digital input.

When a microcontroller reads digital input, the input is determined as either **HIGH or LOW**.

**Determination Criteria:**

Generally, if the input voltage entering the digital input pin is greater than the microcontroller's threshold voltage, it becomes HIGH; if smaller, it becomes LOW.

**Voltage Level Compatibility:**

Connecting a 3.3V digital output to a 5V input is fine (because 3.3V HIGH voltage is greater than the 2.5V threshold voltage).

However, if a 3.3V digital input does not have 5V tolerance capability, you should not connect a 5V digital output to that digital input.

## Pin Modes

### 1. Input Modes

#### 1-1. Floating

**Easy Understanding:**

Like a balloon floating in air. Even a slight wind (noise) makes it wobble around.
- Nothing connected → value randomly switches between 0 and 1
- Like a TV antenna picking up only surrounding noise instead of broadcast signals

**Technical Explanation:**

A state where the pin is not connected to either power (VCC) or ground (GND), commonly called a "floating" state.

In this state, even the slightest static electricity (noise) can cause the value to randomly jump between 0 and 1. It's like an antenna absorbing all surrounding electromagnetic waves.

This is **High Impedance (Hi-Z)**, meaning it's electrically close to isolation. Since almost no current flows through the pin, it doesn't affect external circuits.

**Problem:**

When input is Open (not connected), the voltage level is undefined, causing MCU malfunction or frequent internal logic switching, leading to increased power consumption.

#### 1-2. Pull-Up

**Easy Understanding:**

Like hanging from the ceiling (VCC) with a spring. Stays at the top (1) when left alone.
- Normal state: Pin = 1 (HIGH)
- When button pressed: Forcibly pulled to ground (GND) → Pin = 0 (LOW)

**Technical Explanation:**

The MCU internally pulls the pin toward the power supply (3.3V/5V) with weak force. When nothing is done, the voltage is high, so the **default value is 1**. The pin becomes 0 only when a switch is pressed to forcibly pull it to GND.

**Internal Circuit:** An internal resistor ($R_{PU}$, typically $30k\Omega \sim 50k\Omega$) is connected between the pin and VCC.

**Active Low:** When a button is pressed, current flows directly to GND without going through the resistor, making the pin voltage 0V.

**Features:**
- Strong against noise, the most commonly used button input method in embedded systems
- Supported by most MCUs

#### 1-3. Pull-Down

**Easy Understanding:**

Pressed to the floor (GND) with a heavy weight. Always stays at the bottom (0) when left alone.
- Normal state: Pin = 0 (LOW)
- When button pressed: Pulled up to VCC → Pin = 1 (HIGH)

**Technical Explanation:**

The MCU internally ties the pin to ground (0V) with weak force, and when nothing is done, the **default value is 0**. When a switch is pressed, it becomes 1, opposite to pull-up.

**Internal Circuit:** An internal resistor ($R_{PD}$) is connected between the pin and GND.

**Active High:** When a button is pressed, VCC voltage is applied to the pin, making it Logic 1.

**Features:**
- Matches human intuition (press = 1), but less commonly used than pull-up in circuit design convention
- Some older MCUs didn't support internal pull-down, but modern MCUs like STM32 support both

### 2. Output Modes

#### 2-1. Push-Pull

**Easy Understanding:**

Like elevator doors with two switches, top and bottom.
- HIGH(1) output: Top door opens → pushes electricity out (PUSH)
- LOW(0) output: Bottom door opens → pulls electricity in (PULL)

Both **actively** create voltage!

**Technical Explanation:**

The most common output method.

Two switches are positioned top and bottom inside the MCU.
- **When outputting HIGH(1):** Turn on the top switch to push VCC current out (PUSH)
- **When outputting LOW(0):** Turn on the bottom switch to pull current in (PULL) and dump it to GND

In other words, it actively moves in both 0 and 1 states.

Switches between 0V and 3.3V (or 5V) very quickly and powerfully, used in general output situations like turning on LEDs or sending signals to other chips.

**Caution:**

Do not connect multiple push-pull outputs to a single line. (If one pushes 5V and another pulls to 0V, a short circuit occurs, burning the chip)

#### 2-2. Open-Drain

**Easy Understanding:**

A one-armed switch that pulls down (0) well but can't push up (1).
- LOW(0) output: Firmly pulls down to ground
- HIGH(1) output: Just releases (requires external pull-up resistor)

**Technical Explanation:**

Good at pulling Down(0) but unable to push UP(1), requiring a resistor.

Only one switch connecting to the bottom (GND) exists inside the MCU.
- **When outputting LOW(0):** Close the switch to pull down to ground (GND)
- **When outputting HIGH(1):** Open the switch. But since there's nothing supplying electricity from above, the pin becomes a floating state with no electricity.

**Why use this method?**

1. **When communicating with devices at different voltages (Level Shifting):**
   If my MCU is 3.3V but the counterpart component operates at 5V, set to open-drain and connect an external pull-up resistor to 5V to safely provide a 5V signal. (Sending 3.3V with push-pull risks the counterpart not recognizing it)

2. **When multiple devices communicate on one line (I2C communication):**
   When multiple devices share one line, someone can pull it down to 0, making the entire line 0 (Wired-AND structure), preventing communication collisions.

### 3. Alternate Function Configuration

**Easy Understanding:**

Think of a pin as a "multipurpose room."
- **GPIO mode:** Empty room directly managed by CPU (just turns lights on/off)
- **AF mode (UART setting):** "From today, this is a communication room!" → Communication specialist (UART hardware) monopolizes the room
- **AF mode (Timer setting):** "From today, this is a clock room!" → Time specialist (Timer hardware) monopolizes the room

Transferring control of the pin from CPU → to specialized hardware!

**Technical Explanation:**

**Alternate Function (AF)** transfers pin control from the CPU to internal devices.

Inside the microcontroller (MCU), there are many **'Peripherals'** besides the CPU:
- UART: Communication
- Timer: Time/PWM
- ADC: Analog measurement
- SPI/I2C: High-speed communication

However, the number of pins on the MCU exterior is limited. (Example: 100 built-in functions but only 48 pins) So one pin must be used for different purposes (Alternate) depending on the situation.

#### 3-1. Operating Principle

Uses a **switch (Multiplexer, abbreviated MUX)** inside the pin.

- **GPIO mode:** Switch connects to [Output Data Register]. When we write 0, 1 with code (CPU), the pin changes.
- **AF mode:** Switch connects to [internal device (e.g., Timer)]. From now on, even if CPU commands with `HAL_GPIO_WritePin`, the pin won't listen. Instead, the internal Timer hardware automatically creates signals by switching 0, 1, 0, 1.

#### 3-2. Usage Examples

| Function | Role | CPU's Role |
|----------|------|------------|
| USART (TX/RX) | Exchanges characters with PC or other chips | Just command "Send 'Hello'!", and the UART hardware automatically controls pins at ultra-high speed to shake voltage and send |
| PWM (Timer) | Motor speed control, LED brightness adjustment | Say "Fire at 50% duty!", and Timer hardware automatically turns the pin on and off repeatedly. (CPU can do other things) |
| SPI / I2C | Read sensor values | Say "Read sensor value", and the communication module automatically sends Clock signals to pins and receives data |

#### 3-3. Configuration Method

MCU pins are like transforming robots - you must clearly define "From today, you do this role!" This is called Pin Map configuration.

Typically, in datasheets for chips like STM32, the functions assigned to each pin are listed in a table.

**Example: Fate of PA9 pin**
- AF0: System function
- AF1: Timer 1 (TIM1)
- AF4: Serial communication (USART1_TX)
- AF7: Another communication (USART1)

### 4. Analog Mode

**Easy Understanding:**

Digital = Black and white photo (either 0 or 1)
Analog = Color photo (0.5V, 1.25V, 2.78V... infinite values)

Analog mode measures "exactly how many volts?" instead of "is it 0 or 1?"

**Technical Explanation:**

A mode that escapes the '0 and 1 binary logic'. Passes the voltage magnitude as-is to the ADC for calculation.

#### 4-1. Schmitt Trigger

In digital input mode, there's a **'Schmitt Trigger'** gatekeeper inside the pin.

The gatekeeper's role is to monitor incoming voltage and make clear judgments like "This is 0!" or "This is 1!" and throw it to the CPU.

However, when set to analog mode, the Schmitt Trigger is completely disconnected. The voltage entering the pin is sent raw, without processing, to the **ADC (Analog-to-Digital Converter)** inside the chip.

**Why disconnect the digital circuit?**

Why not just keep it connected and use the ADC? There are two very important reasons to disconnect the digital circuit.

**1. Power Saving:**

Digital circuits (circuits that distinguish between 0 and 1) hate **ambiguous voltages (Middle Voltage)** the most.
- They're comfortable when 0V or 3.3V comes in.
- But what if an ambiguous voltage like 1.65V comes in?

The Schmitt Trigger gets confused asking "Is this 0 or 1?" During this process, internal transistors turn on and off frantically, or stay half-open, causing current to leak (Leakage Current). This drains the battery quickly.

**2. Noise Reduction:**

Subtle electrical vibrations (switching noise) generated when digital circuits operate can interfere with very sensitive analog measurements. So the connection is completely cut off.

#### 4-2. When to Use?

Use when you want to handle **'continuous values'** through pins.

**Voltage Measurement (ADC Input):**
- Temperature sensors, light sensors, battery level checking, etc.
- When you want to know **"Is it 1.25V? Or 2.78V?"** instead of "Is it 1 or 0?"

**Waveform Output (DAC Output):**
- When sending sound through audio jack or creating smooth voltage curves

## Configuration

### 1. Speed / Slew Rate

#### 1-1. Easy Understanding

**Car Acceleration Pedal Analogy:**
- **Low Speed:** Slow, smooth acceleration (good fuel economy, low noise)
- **High Speed:** Rapid acceleration (fast but noisy and shaky)

Same with GPIO! Setting that determines how quickly to raise voltage from 0V → 3.3V.

```
Low Speed:   0V ___/‾‾‾‾ 3.3V  (gentle slope)
High Speed:  0V ___|‾‾‾‾ 3.3V  (vertical cliff)
```

#### 1-2. Definition

Technically refers to **the slope of voltage rise ($dV/dt$) when the output signal changes from 0(0V) to 1(3.3V)**.

- **High Speed:** Voltage rises like a vertical cliff in an instant. Communication is fast but noise, ringing, and overshoot may occur.
- **Low Speed:** Voltage rises diagonally like a gentle slope. Stable but takes time.

From the MCU's perspective, 'Speed' setting determines **"How hard (quickly) should I push up this pin's voltage?"** It's not about frequency itself, but how sharp to make the signal's 'edge'.

#### 1-3. Common Standards

**Low Speed (default):**
- Usage: LED control, button input, general switch control
- Use this for most cases. It's safe and quiet.

**Medium/High Speed:**
- Usage: When exchanging signals millions of times per second like SPI, I2C communication
- Must switch quickly to prevent data corruption.

**Very High Speed:**
- Usage: Ultra-high-speed communication (memory interface, etc.) requiring tens of MHz
- Use only when necessary, in limited cases

### 2. Drive Strength

#### 2-1. Easy Understanding

**Water Pipe Analogy:**
- **Low Drive:** Thin water pipe → water flows in small amounts (weak current)
- **High Drive:** Thick water pipe → water flows strongly (strong current)

Same with GPIO pins - setting how much current can be "pushed out"!

Turning on 1 LED: Thin pipe (Low Drive) is enough
Turning on 10 LEDs: Thick pipe (High Drive) needed

#### 2-2. Definition

When a GPIO pin is in output mode, it means the maximum amount of current that can be sourced or sunk while maintaining the voltage level (Logic High/Low).

It's not simply "sending current", but **"setting the limit of load that can withstand without breaking the defined voltage range ($V_{OH}, V_{OL}$)"**.

- $V_{OH}$ (Output High Voltage): Minimum voltage to be recognized as High (e.g., 2.4V or higher)
- $V_{OL}$ (Output Low Voltage): Maximum voltage to be recognized as Low (e.g., 0.4V or lower)

#### 2-3. Circuit Principle (Internal Mechanism)

**Change in Internal Resistance ($R_{DS(on)}$):**

There's a transistor (MOSFET) inside the MCU responsible for output. Drive Strength setting is like adjusting this transistor's internal resistance value.

- **Low Drive Strength:** High internal transistor resistance. (Narrow pipe → less current flows)
- **High Drive Strength:** Low internal transistor resistance. (Wide pipe → more current flows)

#### 2-4. Problems and Considerations

**Voltage Drop:**

If the load requires a lot of current (e.g., bright LED, motor driver input) but set to Low Drive, what happens?

→ According to Ohm's law ($V=IR$), voltage is consumed at the MCU's internal resistance.

As a result, though commanded to output 3.3V, the actual voltage going out of the pin drops to about 2.5V. Then the connected counterpart component may not recognize the signal, asking "Is this really 3.3V (High)? 2.5V is ambiguous."

**Fan-out Insufficiency:**

When multiple gates (input pins of other chips) need to be connected in parallel to one pin, insufficient driving capability means signals won't be transmitted.

**Excessive Current and Noise (High Setting Side Effect):**

If unnecessarily set to High Drive?

→ Ground Bounce noise occurs during switching moments, shaking signals on adjacent pins. Also increases MCU's heat generation and power consumption.

#### 2-5. Application Guide

**Check Datasheet:**
- Check how many mA the component to connect requires.
- Generally, 2mA~4mA is sufficient for regular signal transmission.

**When Driving LEDs:**
- LEDs consume a lot of current (usually 5~20mA), so set to Medium or High according to calculated resistance value for bright lighting.

**General Communication:**
- Most cases, Low or Medium is sufficient.
- Always setting to High is an amateur-like hardware configuration.

## 2-3. Pin Remapping

 1. Easy Understanding (Analogy)
    Think of MCU as an apartment building.
    
    - **Internal peripherals (UART, Timer, etc.)**: Residents living in each unit (101, 102...)
    - **Pin**: Front door to exit the building
    - **Pin Remapping**: Changing which front door residents use
    
    For example:
    - Normal: UART exits through front door #1. (default setting)
    - Problem: "Oh? Someone else (Timer) also needs to exit through front door #1?"
    - Solution: "Then UART, please exit through front door #2!" (Remapping setting)
    
    **Core concept**: The residents (functions) inside remain the same, only changing which door they exit through!

 2. Definition
    The ability to change the physical pin position where internal peripheral (Peripheral: UART, Timer, SPI, etc.) signals connect.
    
    All peripherals have a 'default pin' set by the manufacturer, but through settings (register manipulation), the signal path can be switched to a pre-designated 'remap pin'.

 3. Operating Mechanism
    Works like a railway switch:
    
    Inside the MCU, there's a **digital switch (Multiplexer, MUX)** that can select signal paths.
    
    ```
    [UART signal] ──┐
                  ├─ switch ──→ Pin #1 (default)
                  └─────────→ Pin #2 (alternate)
    ```
    
    When you set a specific bit to 1 in a register (AFIO, etc.) in code, the internal switch "clicks" and changes, making the signal exit through a different pin.
    
    **Important**: You can't move to any pin arbitrarily! Can only select from "alternate pin" options pre-defined by the manufacturer.

 4. When to Use?
    4-1. When Pin Conflict Occurs
        ```
        Problem: Both UART and Timer want to use PA9 pin.
        Solution: Remap UART to PB6.
        Result: Both can be used simultaneously!
        ```
    
    4-2. When PCB Wiring is Complex
        During circuit board design, if connecting with default pins is difficult, move to another pin that's easier to wire.
        
        (Like opening a detour when roads are blocked)

 5. Actual Example (STM32 based)
    ```c
    // Change UART1's TX pin from default (PA9) to alternate (PB6)
    __HAL_AFIO_REMAP_USART1_ENABLE();  // Enable remapping
    
    // Now UART1 TX signals exit through PB6 instead of PA9!
    ```

## 2-4. EXTI (External Interrupt)

 1. Easy Understanding (Analogy)
    **Doorkeeper (CPU) and Doorbell (EXTI) Analogy**
    
    ### Polling Method = Keep Checking at the Door
    ```c
    while(1) {
        if (button_pressed?)    // Check every 1 second
            greet_guest();
        else
            do_other_work();     // But must keep checking
        delay(1second);
    }
    ```
    → CPU keeps asking "Button pressed? Button pressed?" repeatedly
    → Might miss if button pressed during the 1-second check interval
    → Wastes power because CPU can't rest
    
    ### Interrupt (EXTI) Method = Run When Doorbell Rings
    ```c
    // CPU does other work normally...
    main() {
        watch_TV();
        cook();
        clean();  // ← While doing this...
    }
    
    // This function automatically executes when button pressed!
    void button_interrupt_handler() {
        // "Ding dong!" doorbell rings
        greet_guest();              // Handle urgently
        // Return to clean()
    }
    ```
    → Hardware automatically notifies CPU the moment button is pressed
    → CPU does other work normally, reacts immediately when notified
    → Won't miss anything and saves power

 2. Definition (Technical Perspective)
    **Asynchronous Event Handling:**
    
    A mechanism that requests an exception to the CPU in hardware when an edge (state change) of an external GPIO pin is detected, regardless of the CPU's program counter (PC) flow.
    
    **Event-Driven:**
    
    While polling performs 'synchronous' periodic status checks in software, EXTI reacts immediately when events occur.

 3. Operating Mechanism
    ### 3-1. Easy Explanation
    **EXTI = System that automatically calls CPU "Hey!" when detecting GPIO pin state changes**
    
    Can set when to say "Hey!":
    ```
    Rising Edge:   When changing 0 → 1 (moment of button press)
    Falling Edge:  When changing 1 → 0 (moment of button release)
    Both:          React to both
    ```
    
    Operating sequence:
    ```
    1. [GPIO Pin] Button pressed! (0→1)
         ↓
    2. [Edge Detector] "Change detected!" → Pending Bit = 1
         ↓
    3. [NVIC] "CPU, urgent matter!" (check priority)
         ↓
    4. [CPU] Stop current work → backup current state → jump to interrupt handler
         ↓
    5. [ISR function] Execute button_interrupt_handler()
         ↓
    6. [CPU] Return to original work
    ```
    
    ### 3-2. Technical Explanation (Hardware Level)
    **Edge Detector:**
    
    Monitors signals entering the GPIO pin, and when detecting set changes (Rising/Falling), latches it and sets 'Pending Bit' to 1.
    
    **NVIC (Nested Vectored Interrupt Controller):**
    
    Interrupt management unit attached next to the CPU core. When multiple interrupts occur simultaneously, mediates **priority** and signals the CPU to "stop what you're doing".
    
    **Context Switch:**
    
    The CPU backs up currently executing register values (Context) to the stack, references the predefined **Interrupt Vector Table**, and jumps to the corresponding ISR (Interrupt Service Routine) function.

 4. Engineering Significance (Why Use?)
    ### 4-1. Real-time Determinism
    **Simply**: Handle "absolutely must not miss" events like car collision detection, emergency stop buttons
    
    **Technically**: In embedded systems (especially automotive ECUs), there are requirements (Hard Real-time) to react within $x$ microseconds to sensor signals.
    
    - **Polling Method**: Reaction speed varies with loop cycle (Jitter occurs)
      ```
      Worst case: Detects only after one loop cycle (possible delay of tens of ms)
      ```
    
    - **EXTI Method**: Guarantees response time by reacting immediately in hardware
      ```
      Usually: Reacts within a few microseconds (1,000+ times faster!)
      ```
    
    ### 4-2. Power Efficiency
    **Simply**: CPU sleeps, wakes only when needed → battery lasts longer
    
    **Technically**: Polling requires CPU to stay Active for monitoring, but EXTI allows keeping CPU in Stop or Sleep mode normally, reducing current consumption to $\mu A$ (microampere) units.
    
    ```
    Polling mode:   CPU always on → consumes tens of mA
    EXTI mode:      CPU mostly power-saving → consumes few μA (1,000x difference!)
                    Wake-up only when interrupt occurs
    ```
    
    → Essential for battery-powered IoT sensors, wearable devices!
    
    ### 4-3. CPU Efficiency
    **Polling**: CPU wastes tens of % of time on simple checks
    **Interrupt**: CPU does other important work (computation, communication), reacts only when needed

 5. Caution: Chattering/Bouncing Phenomenon
    **Physical Limitation of Mechanical Buttons**
    
    Users think they press the button once "click!", but actually:
    
    ```
    [Ideal signal]: 0 ───────→ 1 (cleanly once)
    
    [Real signal]:   0 ─┐ ┌┐┌┐┌┐─→ 1 (shakes dozens of times for 10ms!)
                    └┘└┘└┘└┘
    ```
    
    **Why does this happen?**
    
    When pressing mechanical switches, due to subtle vibrations (elastic oscillation) when metal contacts touch, dozens of Rising/Falling Edges repeat for 10ms.
    
    **Problem:**
    
    EXTI is so fast and precise in hardware that it detects all these dozens of shakes, calling ISR dozens of times, causing critical errors.
    
    ```
    User intent: Button clicked once
    Actual operation: 30 interrupts occur!
    ```
    
    **Solution (Debouncing):**
    
    1. **Hardware Method**: RC filter circuit
       - Use resistor (R) and capacitor (C) to smoothly change voltage
       - Circuit physically absorbs vibrations
    
    2. **Software Method**: Time-based filtering
       ```c
       volatile uint32_t last_interrupt_time = 0;
       
       void button_interrupt_handler() {
           uint32_t now = millis();
           
           // Ignore if within 50ms of last interrupt (consider as vibration)
           if (now - last_interrupt_time < 50) {
               return;  // Ignore!
           }
           
           last_interrupt_time = now;
           handle_button();  // Process only real button clicks
       }
       ```

 6. Actual Usage Example
    ```c
    // Toggle LED (PA5) when button (PA0) is pressed
    
    // Initial setting
    attachInterrupt(PA0, button_pressed, RISING);  // Connect interrupt to PA0 rising edge
    
    // CPU normally does other work...
    void loop() {
        // Read sensors, process communication, calculations, etc...
    }
    
    // Automatically executes when button pressed!
    void button_pressed() {
        digitalWrite(PA5, !digitalRead(PA5));  // LED toggle
    }
    ```

## 3. GPIO Operations

### 3-1. Reading Input State

#### Easy Understanding

**Judge and Evidence Analogy:**

Reading GPIO input is like a "courtroom judge" making a verdict (0 or 1) after viewing evidence (voltage).

```
Evidence (voltage)    Judge's (Schmitt Trigger) decision
─────────────────────────────────────
0.5V                 "Definitely 0!" → 0
1.5V                 "Ambiguous... maintain previous verdict"
2.5V                 "Definitely 1!" → 1
```

**Key Point:**
- When middle voltage (gray zone) comes in, postpone judgment and maintain previous state
- Thanks to this, values remain stable even if voltage shakes from noise!

```c
int button_state = digitalRead(12);  // Read pin 12
if (button_state == HIGH) {
    // Button pressed!
}
```

#### 1. Definition

**Digital Read:**

The process where the MCU recognizes **physical voltage (Analog Voltage)** applied to a GPIO pin by converting it to a **logical value (0 or 1)** according to predefined criteria.

**IDR (Input Data Register):**

Inside the MCU, there's an 'input data register' where the current state of pins is stored in real-time. When we give a Read command in code, we're not directly looking at the pin but reading the **value (bit) of this IDR memory address**.

#### 2. Operating Mechanism: Schmitt Trigger

**Easy Explanation:**

Computers must always use binary logic (either 0 or 1). But real-world voltages have many ambiguous middle values.

```
Problem: 3.3V is '1', 0V is '0', but... what about 1.5V?
```

**Schmitt Trigger = Strict Judge:**

A judge standing at the MCU entrance makes decisions according to these rules:
- "Voltage must **definitely** rise above 2.0V to be recognized as '1'." ($V_{IH}$)
- "Voltage must **definitely** fall below 0.8V to be recognized as '0'." ($V_{IL}$)
- In the ambiguous zone between them (0.8V ~ 2.0V), **"postpone judgment (maintain previous state)"**

**Technical Explanation:**

Voltage ambiguity: While 3.3V being '1' and 0V being '0' is clear, the problem is whether to judge 1.5V as 1 or 0.

The Schmitt Trigger solves this problem:

```
V_IH (Input High Threshold): Recognized as '1' if above this voltage (e.g., 2.0V)
V_IL (Input Low Threshold):  Recognized as '0' if below this voltage (e.g., 0.8V)
```

**Hysteresis:**

In the ambiguous voltage zone (0.8V ~ 2.0V), it declares **"postpone judgment (maintain previous state)"**.

Thanks to this, even if voltage shakes around 1.5V due to noise, the MCU can consistently maintain 0 or 1.

```
Voltage change:  0V → 1.5V → 2.5V → 1.5V → 0.5V
Reading result:  0  →  0   →  1   →  1   →  0
                (maintain) (change) (maintain) (change)
```

#### 3. Reasons and Practical Points

**3-1. Sensing:**

The only means to determine whether a button is pressed or a sensor detected an object.

```c
if (digitalRead(BUTTON_PIN) == LOW) {  // Active Low button
    turn_on_LED();
}
```

**3-2. Masking Required:**

Usually reading the IDR register brings in 16 pins' (0~15) states at once as 16-bit data.

```
Example: IDR = 0000 0000 0000 0101 (binary)
    → Pin 0 = 1, Pin 2 = 1, others = 0
```

If only curious about 'pin 3', use C's **bitwise operator (&)** to mask (cover) other bits and extract only bit 3.

**Code Example:**

```c
// Check only pin 3's state
if ((GPIOx->IDR & (1 << 3)) != 0) {
    // If bit 3 is 1?
    printf("Pin 3 is HIGH state\n");
}

// Bit masking operation principle:
// IDR = 0000 0000 0000 1101 (original)
// (1<<3) = 0000 0000 0000 1000 (mask)
// Result = 0000 0000 0000 1000 (extracted only bit 3)
```

**3-3. Application Example:**

```c
// Read multiple buttons simultaneously
uint16_t all_pins = GPIOx->IDR;  // Read all pins at once

bool button1 = (all_pins & (1 << 0)) != 0;  // Pin 0
bool button2 = (all_pins & (1 << 1)) != 0;  // Pin 1
bool button3 = (all_pins & (1 << 2)) != 0;  // Pin 2

// Check multiple pins with one register read (efficient!)
```

### 3-2. Output Set / Clear

#### Easy Understanding

**Light Switch Analogy:**

Controlling GPIO output is just like turning light switches on and off at home.

```
Set:    Switch ON  → Light on (3.3V output)
Clear:  Switch OFF → Light off (0V output)
```

**Actual Usage:**

```c
digitalWrite(13, HIGH);  // Turn on pin 13 (Set)
digitalWrite(13, LOW);   // Turn off pin 13 (Clear)
```

This turns on LEDs, spins motors, activates relays!

#### 1. Definition

**Set:**

Making the GPIO pin's voltage level Logic High (1, VCC, 3.3V) state.

**Clear:**

Making the GPIO pin's voltage level Logic Low (0, GND, 0V) state.

Internally in the MCU, it's the act of writing 1 or 0 to a specific bit of a memory address called **ODR (Output Data Register)**.

#### 2. Operating Mechanism

**Easy Explanation: Switch Operation**

**Set: "Turn on the light switch!"**

Current starts flowing in the circuit connected to the pin. (LED turns on, motor rotates)

```c
PORTA |= (1 << 3);  // Make only pin 3 to 1
```

**Clear: "Turn off the light switch!"**

Current in the circuit connected to the pin is cut off. (LED turns off, motor stops)

```c
PORTA &= ~(1 << 3);  // Make only pin 3 to 0
```

**Technical Explanation: Read-Modify-Write Process**

Looking at the above code (`|=`, `&=`), it's not simply commanding "turn on pin 3".

It actually goes through 3 steps:

```
1. Read:    Read current state of all pins
   PORTA = 0000 0101 (currently pins 0, 2 are on)

2. Modify:  Change only bit 3 to 1
   0000 0101 | 0000 1000 = 0000 1101

3. Write:   Overwrite the modified value back to the register
   PORTA = 0000 1101 (now pins 0, 2, 3 are on)
```

**Caution:**

Why is this complex process a problem?

If an interrupt intervenes between Read and Write to change another pin, that change can be lost! (Race Condition)

This problem is solved by **Atomic Operations**. (explained in next section)

#### 3. Reasons and Practical Points

**3-1. Actuation:**

The basis of all physical control like LED control, relay operation, LCD screen turning on.

```c
// LED blinking
digitalWrite(LED_PIN, HIGH);  // Turn on
delay(1000);
digitalWrite(LED_PIN, LOW);   // Turn off
delay(1000);
```

**3-2. Initialization:**

Pins shouldn't operate randomly as soon as the MCU turns on. (e.g., motor suddenly spinning is dangerous)

So in the Configuration stage, **Initial State** must be clearly set to either Set(1) or Clear(0) beforehand.

```c
void setup() {
    pinMode(MOTOR_PIN, OUTPUT);
    digitalWrite(MOTOR_PIN, LOW);  // Initial value: start in off state!
    
    pinMode(LED_PIN, OUTPUT);
    digitalWrite(LED_PIN, LOW);    // LED also starts in off state
}
```

**3-3. Practical Example:**

```c
// Control air conditioner with relay
#define AC_RELAY_PIN 7

void turn_on_AC() {
    digitalWrite(AC_RELAY_PIN, HIGH);  // Set
    Serial.println("AC ON");
}

void turn_off_AC() {
    digitalWrite(AC_RELAY_PIN, LOW);   // Clear
    Serial.println("AC OFF");
}

// Direct control with bit operations (advanced)
void control_multiple_pins() {
    // Turn on pins 3, 5, 7 simultaneously
    PORTA |= (1 << 3) | (1 << 5) | (1 << 7);
    
    // Turn off pins 2, 4 simultaneously
    PORTA &= ~((1 << 2) | (1 << 4));
}
```

### 3-3. Toggle Operations

#### Easy Understanding

**Click Pen Analogy:**

Toggle is like pressing a "click pen"!

```
No need to check current state, just press:
- If tip is out → goes in
- If tip is in → comes out
```

**Set/Clear vs Toggle:**

```c
// Set/Clear: explicitly specify state
digitalWrite(13, HIGH);  // "On!"
digitalWrite(13, LOW);   // "Off!"

// Toggle: opposite of current state
toggle(13);  // "Do opposite!"
```

**Actual Usage:**

```c
// LED blinking
while(1) {
    toggle(LED_PIN);  // Repeatedly on ↔ off
    delay(500);
}
```

#### 1. Definition

An operation that inverts the current state of a pin.

If currently High(1), changes to Low(0); if Low(0), changes to High(1).

In C language, implemented using the **XOR operator (^)**.

```c
PORTA ^= (1 << 3);  // Invert value of pin 3
```

#### 2. Operating Mechanism

**Easy Explanation: Click Pen**

**Set/Clear:** Commanding state like "Extend the pen tip!" or "Retract the pen tip!".

**Toggle:** "Press the pen once!"
- If the tip is out now, it will go in; if in, it will come out.
- You just press (Action) without checking the current state, and the state reverses.

**Technical Explanation: XOR Bit Operation**

Toggle uses XOR (exclusive OR) operation:

```
XOR truth table:
Current  XOR  1   =  Result
  0    XOR  1   =   1   (inverted)
  1    XOR  1   =   0   (inverted)
```

**Operating Process:**

```c
// Toggle pin 3
PORTA ^= (1 << 3);

// Internal operation:
// 1. Read:   PORTA = 0000 0101 (currently pins 0, 2 on)
// 2. XOR:    0000 0101 ^ 0000 1000 = 0000 1101
// 3. Write:  PORTA = 0000 1101 (pin 3 inverted)
```

**Caution:**

Toggle also goes through "read current state (Read) → invert (Modify) → write again (Write)".

So toggling at very high speed can cause **RMW (Read-Modify-Write) problems**. (Race Condition)

#### 3. Reasons and Practical Points

**3-1. Heartbeat:**

Most commonly used to blink an LED every 1 second to check if embedded equipment is alive or dead. (continuously toggle in infinite loop)

```c
// System operation confirmation LED
void loop() {
    // ... main tasks ...
    
    // Toggle LED every 1 second (proof system is alive)
    static uint32_t last_toggle = 0;
    if (millis() - last_toggle >= 1000) {
        digitalWrite(HEARTBEAT_LED, !digitalRead(HEARTBEAT_LED));
        last_toggle = millis();
    }
}
```

**3-2. Clock Generation (Square Wave):**

Continuously toggling at constant speed creates a **square wave**. Can supply clock signals to other chips or make buzzer sounds.

```c
// Create 1kHz buzzer sound
void beep_1kHz() {
    for (int i = 0; i < 1000; i++) {  // For 1 second
        digitalWrite(BUZZER_PIN, HIGH);
        delayMicroseconds(500);  // 0.5ms
        digitalWrite(BUZZER_PIN, LOW);
        delayMicroseconds(500);  // 0.5ms
        // Total 1ms period = 1kHz
    }
}

// Or use toggle
void beep_1kHz_toggle() {
    for (int i = 0; i < 2000; i++) {  // 2000 toggles = 1 second
        toggle(BUZZER_PIN);
        delayMicroseconds(500);
    }
}
```

**3-3. Concise Code:**

Using Toggle eliminates the need for code to check current state, making it more concise.

```c
// Without Toggle (complex)
if (digitalRead(LED_PIN) == HIGH) {
    digitalWrite(LED_PIN, LOW);
} else {
    digitalWrite(LED_PIN, HIGH);
}

// Using Toggle (concise)
digitalWrite(LED_PIN, !digitalRead(LED_PIN));

// Or direct bit operation
PORTA ^= (1 << LED_BIT);
```

**3-4. Practical Example:**

```c
// Invert LED state every time button is pressed
void button_interrupt_handler() {
    digitalWrite(LED_PIN, !digitalRead(LED_PIN));  // Toggle
    // If on, turns off; if off, turns on
}

// Debugging pin toggle (measure timing with oscilloscope)
void measure_execution_time() {
    digitalWrite(DEBUG_PIN, HIGH);  // Start marker
    
    complex_calculation();  // Code to measure
    
    digitalWrite(DEBUG_PIN, LOW);   // End marker
    // Measure HIGH section time with oscilloscope
}
```

### 3-4. Atomic Bit Operations

#### Easy Understanding

**Bank Account Deposit/Withdrawal Analogy:**

**Dangerous Method (RMW):**
```
1. Check account balance (Read):      10,000 won
2. Calculate (Modify):                 10,000 + 5,000 = 15,000 won
3. Write to account (Write):           15,000 won

Problem: If someone intervenes during step 2 to withdraw 3,000 won?
     → That change disappears!
```

**Safe Method (Atomic):**
```
Tell the bank "Deposit 5,000 won!" Done in one command.
→ Bank handles it without interruption in between
```

**In GPIO:**
```c
// Dangerous: Read-Modify-Write
PORTA |= (1 << 3);  // Read → modify → write (3 steps, room for interruption)

// Safe: Atomic
BSRR = (1 << 3);    // Just write once! (1 step, no room for interruption)
```

#### 1. Definition

**Atomic:**

Means "cannot be divided further". In programming, it means **"an instruction that completes in one go without being interrupted in the middle"**.

**BSRR (Bit Set/Reset Register):**

Instead of reading and modifying the general ODR (output register), writing 1 to the **'dedicated setting register (BSRR)'** makes the hardware immediately change that pin automatically.

A hardware function to control pins with **Write Only**, without the Read-Modify-Write (RMW) process.

#### 2. Operating Mechanism: RMW Tragedy vs Atomic Solution

**Situation Setup:**

Assume pins 0 and 1 are both in state 0 (00).

**[Dangerous Method] RMW (Read-Modify-Write)**

When executing `PORTA |= (1 << 0);` (turn on pin 0) in C, the CPU actually does 3 steps.

```
1. Read:
   Reads current port state (00) and brings it to CPU.
   
2. Modify:
   Changes bit 0 of the brought value to 1. (01)
   
3. Write:
   Writes the changed value (01) back to the port register.
```

**Problem Occurrence (Race Condition):**

```
Time sequence:
─────────────────────────────────────────────
Main code:    1. Read (read 00)
Main code:    2. Modify (modifying to 01...)
                    ↓
            [Interrupt occurs!]
Interrupt:         Turn on pin 1 (port = 10)
            [Interrupt ends]
                    ↓
Main code:    3. Write (write the previously calculated 01)
─────────────────────────────────────────────
Result: Port = 01 (Pin 1 turned on by interrupt is off!)
Expected: Port = 11 (Both should be on)
```

**Data loss occurs!** Pin 1 that was turned on by the interrupt turns off again.

**[Safe Method] Atomic Operation (BSRR)**

```c
BSRR = (1 << 0);  // Write 1 to bit 0
```

**Operation:**
- "Don't care what the current state is. Just turn on pin 0!"
- CPU completes the command with **a single instruction (Store)**.
- No room for interrupts to intervene in the middle.

```
Time sequence:
─────────────────────────────────────────────
Main code:    Write (1<<0) to BSRR [atomic execution, cannot be interrupted]
                    ↓
            [Even if interrupt comes at this moment, doesn't matter]
Interrupt:         Turn on pin 1 (operates independently)
─────────────────────────────────────────────
Result: Port = 11 (Both normally on!)
```

**STM32 BSRR Register Structure:**

```
BSRR[31:16] = Reset bits (writing 1 Clears that pin)
BSRR[15:0]  = Set bits (writing 1 Sets that pin)

Example:
GPIOA->BSRR = (1 << 3);   // Set pin 3
GPIOA->BSRR = (1 << 19);  // Clear pin 3 (16 + 3)
```

#### 3. Reasons and Practical Points

**3-1. Preventing Race Condition:**

As explained above, prevents data corruption when multiple settings (multithreading, interrupts) simultaneously touch pins.

**Essential** in **RTOS (Real-Time Operating System)** or complex firmware, not optional.

```c
// In main loop
void main_loop() {
    GPIOA->BSRR = (1 << 5);  // Turn on LED (atomic)
}

// In interrupt handler
void timer_interrupt() {
    GPIOA->BSRR = (1 << 6);  // Turn on different LED (atomic)
    // No mutual interference!
}
```

**3-2. Execution Speed:**

Writing in 1 step is much faster than reading, modifying, and writing in 3 steps!

```
RMW method:   Read(1cycle) + Modify(1cycle) + Write(1cycle) = 3cycles
Atomic:       Write(1cycle) = 1cycle

Speed: 3 times faster!
```

**3-3. Code Comparison:**

```c
// [Bad Example] RMW - Race Condition risk
void set_pin_unsafe() {
    GPIOA->ODR |= (1 << 3);  // Dangerous!
}

// [Good Example] Atomic - Safe
void set_pin_safe() {
    GPIOA->BSRR = (1 << 3);  // Safe!
}

// [Good Example] Control multiple pins atomically too
void set_multiple_pins() {
    GPIOA->BSRR = (1 << 3) | (1 << 5) | (1 << 7);  // Set pins 3, 5, 7 simultaneously
}

// [Good Example] Set and Clear simultaneously
void set_and_clear_pins() {
    GPIOA->BSRR = (1 << 3) |      // Set pin 3
                  (1 << (5+16));  // Clear pin 5
    // Two operations in one command!
}
```

**3-4. Essential in RTOS Environment:**

```c
// Two tasks running simultaneously in FreeRTOS
void Task1(void *param) {
    while(1) {
        GPIOA->BSRR = (1 << 0);  // Turn on pin 0 (atomic)
        vTaskDelay(100);
    }
}

void Task2(void *param) {
    while(1) {
        GPIOA->BSRR = (1 << 1);  // Turn on pin 1 (atomic)
        vTaskDelay(100);
    }
}
// Thanks to BSRR, operates safely without mutual interference!
```

**3-5. Caution:**

Not all MCUs support BSRR:
- **STM32:** Supports BSRR ✓
- **AVR (Arduino Uno, etc.):** No BSRR → solve by disabling interrupts
  ```c
  cli();  // Turn off interrupts
  PORTA |= (1 << 3);  // Manipulate safely
  sei();  // Turn on interrupts
  ```
- **ARM Cortex-M:** Supports BSRR or similar atomic instructions

## 4. GPIO Registers & Configuration

### 4-1. ODR vs IDR vs BSRR Register Comparison

#### Easy Understanding

**Post Office Analogy:**

```
IDR (Input Data Register):    Check mailbox (read-only)
ODR (Output Data Register):   Send mail (read/write)
BSRR (Bit Set/Reset Register): Express delivery (write-only, fast!)
```

#### Register Comparison Table

| Register | Purpose | Access Method | Speed | Safety | When to Use |
|---------|---------|--------------|-------|--------|-------------|
| **IDR** | Read input | Read Only | Normal | Safe | Check button, sensor state |
| **ODR** | Output control | Read/Write | Slow | Dangerous (RMW) | Simple output (without interrupts) |
| **BSRR** | Output control | Write Only | Fast | Safe (Atomic) | Multitasking, interrupt environment |

#### Detailed Explanation

**1. IDR (Input Data Register)**

```c
uint16_t value = GPIOA->IDR;  // Read all pin states of port A

// Check specific pin only
if (GPIOA->IDR & (1 << 3)) {
    // Pin 3 is HIGH
}
```

- **Feature:** Read-only
- **Purpose:** Check GPIO input state
- **Note:** Write attempts are ignored

**2. ODR (Output Data Register)**

```c
GPIOA->ODR = 0x00FF;  // Set all lower 8 bits to 1

// Set specific bit (RMW method)
GPIOA->ODR |= (1 << 3);   // Set pin 3
GPIOA->ODR &= ~(1 << 5);  // Clear pin 5
```

- **Feature:** Read/write possible
- **Risk:** Race Condition possible with Read-Modify-Write
- **Purpose:** Simple environment (main loop without interrupts)

**3. BSRR (Bit Set/Reset Register)**

```c
// STM32 structure:
// BSRR[15:0]  = Set bits (writing 1 makes that pin High)
// BSRR[31:16] = Reset bits (writing 1 makes that pin Low)

GPIOA->BSRR = (1 << 3);      // Set pin 3
GPIOA->BSRR = (1 << (5+16)); // Reset pin 5

// Simultaneous Set/Reset
GPIOA->BSRR = (1 << 3) | (1 << (5+16));  // Set pin 3, Reset pin 5
```

- **Feature:** Write-only, atomic execution
- **Advantage:** No Race Condition, fast speed
- **Purpose:** RTOS, interrupt environment (essential!)

#### Performance Comparison

```c
// Speed measurement (STM32F4 basis)
// ODR method (RMW):     ~3 cycles
// BSRR method (Atomic): ~1 cycle

// Safety:
// ODR:  Dangerous in interrupt environment ⚠️
// BSRR: Always safe ✓
```

### 4-2. Full Port Read/Write

#### Easy Understanding

**Traffic Light Analogy:**

Individual control: Turn on red, yellow, green one by one
Port control: "Red+yellow off, green on" all at once!

#### Purpose and Usage

**Full Port Read:**

```c
// Read 16 pin states at once
uint16_t port_state = GPIOA->IDR;

// Bit analysis
bool pin0 = (port_state & (1 << 0)) != 0;
bool pin1 = (port_state & (1 << 1)) != 0;
// ...

// Pattern check
if ((port_state & 0x0F) == 0x05) {  // Check if lower 4 bits are 0101
    // Specific pattern detected
}
```

**Full Port Write:**

```c
// Set all pins at once
GPIOA->ODR = 0xAAAA;  // 1010 1010 1010 1010 pattern

// Or safely with BSRR
GPIOA->BSRR = 0xAAAA0000;  // Lower 16 bits Set, upper Reset
```

**Practical Application Example:**

```c
// 8-bit parallel data bus (LCD control)
void write_parallel_data(uint8_t data) {
    // Output data to PA0~PA7
    uint16_t temp = GPIOA->ODR;
    temp &= 0xFF00;  // Keep upper 8 bits
    temp |= data;     // Data in lower 8 bits
    GPIOA->ODR = temp;
}

// 7-Segment Display (control multiple pins simultaneously)
const uint8_t digit_patterns[10] = {
    0x3F, 0x06, 0x5B, 0x4F, 0x66,  // 0~4
    0x6D, 0x7D, 0x07, 0x7F, 0x6F   // 5~9
};

void display_digit(uint8_t num) {
    if (num > 9) return;
    GPIOB->ODR = digit_patterns[num];  // Output pattern at once
}
```

### 4-3. GPIO Initialization Sequence

#### Easy Understanding

**Car Ignition Sequence Analogy:**

```
1. Insert key         → Activate clock (power supply)
2. Check gear        → Set mode (input/output)
3. Adjust steering   → Set speed/drive
4. Start engine      → Set initial value
```

Just like ignition won't work if the sequence is wrong, GPIO sequence is important too!

#### Standard Initialization Sequence

**Step 1: Activate Clock**

```c
// STM32 example
RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;  // Turn on GPIOA clock

// Wait for clock to stabilize (usually unnecessary but safe)
volatile uint32_t dummy = RCC->AHB1ENR;  // Read back
```

**Step 2: Mode Setting (Input/Output/AF/Analog)**

```c
// Set PA3 as output
GPIOA->MODER &= ~(0x3 << (3*2));  // Clear corresponding bits
GPIOA->MODER |= (0x1 << (3*2));   // 01 = output mode
```

**Step 3: Output Type Setting (Push-Pull/Open-Drain)**

```c
GPIOA->OTYPER &= ~(1 << 3);  // 0 = Push-Pull
// GPIOA->OTYPER |= (1 << 3);  // 1 = Open-Drain
```

**Step 4: Speed Setting**

```c
GPIOA->OSPEEDR &= ~(0x3 << (3*2));
GPIOA->OSPEEDR |= (0x2 << (3*2));  // 10 = High Speed
```

**Step 5: Pull-up/Pull-down Setting**

```c
GPIOA->PUPDR &= ~(0x3 << (3*2));
GPIOA->PUPDR |= (0x1 << (3*2));  // 01 = Pull-up
```

**Step 6: Set Initial Output Value**

```c
GPIOA->BSRR = (1 << (3+16));  // Start with initial value LOW (safe)
```

#### Complete Initialization Function Example

```c
void GPIO_Init_PA3_Output(void) {
    // 1. Activate clock
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
    
    // 2. Mode: Output
    GPIOA->MODER &= ~(0x3 << (3*2));
    GPIOA->MODER |= (0x1 << (3*2));
    
    // 3. Type: Push-Pull
    GPIOA->OTYPER &= ~(1 << 3);
    
    // 4. Speed: Medium
    GPIOA->OSPEEDR &= ~(0x3 << (3*2));
    GPIOA->OSPEEDR |= (0x1 << (3*2));
    
    // 5. Pull-up/Pull-down: None
    GPIOA->PUPDR &= ~(0x3 << (3*2));
    
    // 6. Initial value: LOW
    GPIOA->BSRR = (1 << (3+16));
}
```

**Cautions:**

- If you don't turn on the clock first, other settings are ignored!
- If you don't set initial value, starts in undefined state (dangerous)
- Writing output value before setting mode is ignored

### 4-4. GPIO Lock Mechanism

#### Easy Understanding

**Padlock Analogy:**

```
Normal setting: Freely open and close doors
Lock setting: Lock with padlock → Can't open until reset!
```

Important pins (motor, safety devices) are locked to prevent accidental setting changes since it's dangerous!

#### Definition

GPIO Lock is a hardware protection feature that **"freezes"** specific pin settings (mode, speed, type, etc.) to prevent changes due to mistakes or bugs.

Once Locked, **cannot be unlocked until MCU reset!**

#### Operating Principle

**Lock Sequence (STM32 basis):**

```c
// Write to LCKR register in specific order
void GPIO_Lock_Pin(GPIO_TypeDef *GPIOx, uint16_t pin) {
    uint32_t temp = GPIO_LCKR_LCKK | (1 << pin);
    
    // Lock sequence (order defined in manual is mandatory!)
    GPIOx->LCKR = temp;           // LCKK=1, set pin bit
    GPIOx->LCKR = (1 << pin);     // LCKK=0
    GPIOx->LCKR = temp;           // LCKK=1
    temp = GPIOx->LCKR;           // Read back (confirm Lock)
    
    // Confirm Lock success
    if (GPIOx->LCKR & GPIO_LCKR_LCKK) {
        // Lock successful!
    }
}
```

**Usage Example:**

```c
void Safety_Critical_Init(void) {
    // 1. Configure motor control pin
    GPIO_Init_Motor_Pin();
    
    // 2. Lock after configuration complete!
    GPIO_Lock_Pin(GPIOA, 5);  // Lock PA5
    
    // 3. Now nobody can change PA5 settings
    // GPIOA->MODER ... attempts are ignored!
}
```

#### When to Use?

**1. Safety-Critical Systems:**
- Automotive ECU (motor, brakes)
- Medical devices (pumps, valves)
- Industrial control (emergency stop)

**2. Certification Systems:**
- Compliance with safety standards like MISRA-C, IEC 61508

**3. Multi-Developer Environment:**
- Prevent accidental important pin setting changes

#### Cautions

```c
// ⚠️ Once Locked, cannot unlock until reset!
GPIO_Lock_Pin(GPIOA, 3);

// All subsequent setting change attempts are ignored
GPIOA->MODER |= (0x3 << (3*2));  // Ignored!
GPIOA->OSPEEDR |= (0x3 << (3*2));  // Ignored!

// Unlock method: Only MCU reset possible
NVIC_SystemReset();
```

### 4-5. Hardware Input Filtering

#### Easy Understanding

**Water Purifier Analogy:**

```
No filter: Muddy water comes in as is (includes noise)
With filter: Only clean water passes (stable signal)
```

GPIO input also filters out electrical noise with hardware filters!

#### Definition

**Hardware Input Filter** is a function that removes short glitches or noise from signals entering GPIO pins in hardware.

Can get clean signals without software debouncing.

#### Filter Types

**1. Digital Filter:**

```c
// Some STM32 models
// Change recognized only if same value read N consecutive times
// (Configurable: 1~255 samples)
```

**2. Analog Filter:**

Some MCUs have built-in RC filters inside pins

**3. Hysteresis:**

Natural filtering through the difference between Schmitt Trigger's $V_{IH}$ and $V_{IL}$

#### Usage Example

```c
// STM32 example (some models)
// Activate filter
GPIOA->AFR[0] |= GPIO_AFRL_AFSEL3_FILTER;  // Activate PA3 filter

// Set filter sample count (pseudo code)
FILTER->SAMPLES = 5;  // Change recognized only if same value 5 consecutive times
```

**Effect:**

```
Filter OFF:
Signal: _____|‾‾|__|‾‾‾‾‾‾  (includes glitches)
Read:   0000011001111111     (incorrect edge detection)

Filter ON:
Signal: _____|‾‾|__|‾‾‾‾‾‾
Read:   000000000111111      (stable)
```

#### Practical Application

**Button Input:**
```c
// Hardware debounces when filter is on
// No need for software delay!
```

**Sensor Signals:**
```c
// Useful in noisy environments (near motors)
```

#### Limitations and Cautions

- Not all MCUs support it (datasheet check mandatory)
- Signal delay occurs due to filter (usually a few microseconds)
- Cannot filter ultra-high-speed signals

## 5. Electrical Characteristics

### Easy Understanding

**Door Height Analogy:**

```
V_IH: "You can enter if above this height" (2.0V)
V_IL: "You must exit if below this height" (0.8V)
```

If voltage is ambiguous → "gray zone" → judgment postponed!

### Key Parameters (Quick Reference)

| Parameter | Meaning | Typical Value (3.3V system) | Importance |
|-----------|---------|----------------------------|------------|
| $V_{IH}$ | Input High minimum voltage | 2.0V | ★★★ |
| $V_{IL}$ | Input Low maximum voltage | 0.8V | ★★★ |
| $V_{OH}$ | Output High minimum voltage | 2.4V | ★★★ |
| $V_{OL}$ | Output Low maximum voltage | 0.4V | ★★★ |
| $I_{OH}$ | Output High maximum current | 4~25mA | ★★ |
| $I_{OL}$ | Output Low maximum current | 4~25mA | ★★ |
| $I_{leak}$ | Input leakage current | < 1μA | ★ |

### Detailed Explanation

**Input Threshold Voltage:**

```
3.3V ─┐
      │
V_IH ─┤─── Above here is definitely HIGH (1)
      │
      │    ← Gray zone (hysteresis)
      │
V_IL ─┤─── Below here is definitely LOW (0)
      │
0V  ──┘
```

**Output Capability:**

```c
// If $I_{OH}$ = 8mA
// 1 LED (needs 20mA) → Cannot directly drive ✗
// 1 LED (5mA)        → Possible ✓
// Multiple in parallel → Not possible ✗
```

### Practical Checklist

```c
// ✓ Things to check in datasheet:
// 1. V_DD range (2.0~3.6V?)
// 2. Is it 5V tolerant pin? (Can connect 5V to 3.3V system?)
// 3. Maximum output current (Can directly drive LED?)
// 4. Total port maximum current (sum of all pins)
```

**Note:** For detailed electrical specifications, refer to `00. Hardware Fundamentals > 0.3 Digital Logic & Interface`.
