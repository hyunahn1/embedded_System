# 1.3 Hardware Interfacing (Peripherals)

## Topics Covered

### GPIO (General Purpose Input/Output)
- **Pin Modes**
  - Input modes: floating, pull-up, pull-down
  - Output modes: push-pull, open-drain
  - Alternate function configuration
  - Analog mode

- **Configuration**
  - Speed/slew rate settings
  - Drive strength
  - Pin remapping
  - External interrupt configuration (EXTI)

- **Operations**
  - Reading input state
  - Setting/clearing output
  - Toggle operations
  - Atomic bit operations

### ADC/DAC (Analog-Digital Converter / Digital-Analog Converter)
- **ADC Fundamentals**
  - Resolution (8-bit, 10-bit, 12-bit, 16-bit)
  - Sampling rate and conversion time
  - Reference voltage selection
  - Single-ended vs differential input

- **ADC Modes**
  - Single conversion
  - Continuous conversion
  - Scan mode (multiple channels)
  - Discontinuous mode

- **Advanced Features**
  - DMA integration for continuous sampling
  - Oversampling and averaging
  - Analog watchdog
  - Calibration procedures

- **DAC Operations**
  - Resolution and update rate
  - Output buffering
  - Waveform generation
  - DMA-driven DAC updates

### Timers
- **Timer Types**
  - Basic timers
  - General-purpose timers
  - Advanced control timers
  - Low-power timers
  - Watchdog timers

- **PWM Generation**
  - PWM mode configuration
  - Duty cycle control
  - Frequency adjustment
  - Center-aligned vs edge-aligned PWM
  - Complementary outputs with dead-time

- **Input Capture**
  - Frequency measurement
  - Pulse width measurement
  - Edge detection
  - Encoder interface mode

- **Watchdog Timer (WDT)**
  - Independent watchdog (IWDG)
  - Window watchdog (WWDG)
  - Reset generation
  - Timeout configuration
  - Debug mode behavior

### Communication Protocols

#### UART/USART (Universal Asynchronous/Synchronous Receiver-Transmitter)
- **Configuration**
  - Baud rate calculation
  - Data bits, parity, stop bits
  - Flow control (RTS/CTS)
  - Synchronous mode

- **Operations**
  - Polling-based transmission/reception
  - Interrupt-driven I/O
  - DMA-based transfers
  - Error detection (framing, overrun, parity)

#### SPI (Serial Peripheral Interface)
- **Configuration**
  - Master vs Slave mode
  - Clock polarity (CPOL) and phase (CPHA)
  - Bit order (MSB/LSB first)
  - Clock speed
  - NSS (chip select) management

- **Operations**
  - Full-duplex communication
  - Multi-slave configuration
  - DMA transfers
  - Hardware NSS control

#### I2C (Inter-Integrated Circuit)
- **Configuration**
  - Master vs Slave mode
  - Standard (100kHz) vs Fast (400kHz) vs Fast+ (1MHz)
  - 7-bit vs 10-bit addressing
  - Clock stretching

- **Operations**
  - Start/stop conditions
  - Address transmission
  - Data read/write
  - Repeated start
  - Multi-master arbitration
  - ACK/NACK handling

- **Advanced Features**
  - DMA support
  - PEC (Packet Error Checking)
  - SMBus compatibility

### DMA (Direct Memory Access)
- **DMA Basics**
  - Memory-to-memory transfers
  - Memory-to-peripheral transfers
  - Peripheral-to-memory transfers

- **Configuration**
  - Channel/stream selection
  - Priority levels
  - Transfer size (byte, half-word, word)
  - Circular mode vs normal mode
  - Memory/peripheral increment

- **Mechanics**
  - Transfer complete interrupts
  - Half-transfer interrupts
  - Error handling
  - Double buffering
  - Burst mode

- **Integration with Peripherals**
  - ADC + DMA for continuous sampling
  - UART + DMA for high-speed communication
  - SPI + DMA for large data transfers
  - Timer + DMA for pattern generation

## Key Concepts

- **Polling vs Interrupt vs DMA**
- **Peripheral clock enabling**
- **Pin multiplexing and alternate functions**
- **Electrical characteristics (voltage levels, current drive)**
- **Signal integrity and noise immunity**

## Practical Exercises

1. Implement a GPIO-based LED blinker with button input
2. Configure ADC with DMA for continuous data acquisition
3. Generate PWM signals with variable duty cycle
4. Implement UART communication with interrupt handling
5. Write SPI driver for external sensor
6. Create I2C master driver for EEPROM access
7. Measure frequency using timer input capture
8. Implement watchdog timer reset mechanism

## Code Examples

### GPIO Configuration (STM32-style)
```c
// Enable clock
RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;

// Configure PA5 as output
GPIOA->MODER &= ~(3 << (5*2));  // Clear
GPIOA->MODER |= (1 << (5*2));   // Set to output mode

// Set output
GPIOA->ODR |= (1 << 5);
```

### Timer PWM Setup
```c
// Configure timer for PWM
TIM2->PSC = 84 - 1;        // Prescaler
TIM2->ARR = 1000 - 1;      // Period
TIM2->CCR1 = 500;          // 50% duty cycle
TIM2->CCMR1 |= TIM_CCMR1_OC1M_1 | TIM_CCMR1_OC1M_2; // PWM mode 1
TIM2->CCER |= TIM_CCER_CC1E;  // Enable output
TIM2->CR1 |= TIM_CR1_CEN;     // Start timer
```

### DMA Setup for ADC
```c
// Configure DMA for ADC
DMA2_Stream0->CR = 0;  // Disable to configure
DMA2_Stream0->PAR = (uint32_t)&ADC1->DR;  // Peripheral address
DMA2_Stream0->M0AR = (uint32_t)adc_buffer; // Memory address
DMA2_Stream0->NDTR = ADC_BUFFER_SIZE;      // Number of data
DMA2_Stream0->CR |= DMA_SxCR_CIRC |        // Circular mode
                    DMA_SxCR_MINC |        // Memory increment
                    DMA_SxCR_MSIZE_0 |     // 16-bit memory
                    DMA_SxCR_PSIZE_0;      // 16-bit peripheral
DMA2_Stream0->CR |= DMA_SxCR_EN;           // Enable DMA
```

## References

- MCU Reference Manuals
- Peripheral datasheets
- Protocol specifications (I2C, SPI, UART standards)
- Application notes for peripheral usage
