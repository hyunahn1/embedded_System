# 6.3 Measurement & Analysis

## Topics Covered

### Oscilloscope & Logic Analyzer Usage

#### Oscilloscope Basics

##### Overview
- **Purpose**: Visualize voltage over time
- **Use Cases**: 
  - Signal integrity analysis
  - PWM verification
  - Analog sensor debugging
  - Power supply ripple measurement
  - Timing analysis

##### Key Specifications
- **Bandwidth**: Maximum frequency (e.g., 100 MHz)
- **Sample Rate**: Samples per second (e.g., 1 GSa/s)
- **Channels**: Number of simultaneous signals (2 or 4 typical)
- **Memory Depth**: How long signal can be captured
- **Resolution**: Vertical resolution (8-bit typical)

##### Basic Operations
```
1. Connect probe to signal
2. Set vertical scale (V/div)
3. Set horizontal scale (time/div)
4. Adjust trigger level and type
5. Capture and analyze
```

##### Triggering
- **Edge Trigger**: Rising/falling edge
- **Pulse Width Trigger**: Specific pulse width
- **Logic Trigger**: Complex logic conditions
- **Serial Trigger**: UART, SPI, I2C patterns

##### Measurements
- **Voltage**: Peak-to-peak, RMS, average
- **Time**: Period, frequency, duty cycle
- **Edge Times**: Rise time, fall time
- **Phase**: Between two signals

##### Example: PWM Analysis
```
Signal: Motor control PWM
- Set to 5V/div (vertical)
- Set to 1ms/div (horizontal)
- Trigger on rising edge
- Measure:
  * Frequency: 20 kHz
  * Duty cycle: 75%
  * Rise time: 100 ns
  * Overshoot: 5%
```

##### Probes
- **1X Probe**: Direct connection, high capacitance
- **10X Probe**: Attenuated, low capacitance (preferred for >1 MHz)
- **Differential Probe**: Measure voltage difference
- **Current Probe**: Measure current

#### Logic Analyzer

##### Overview
- **Purpose**: Capture digital signals (high/low)
- **Channels**: Many channels (8, 16, 32+ typical)
- **Sample Rate**: Very high (100+ MS/s)
- **Use Cases**:
  - Protocol debugging (I2C, SPI, UART)
  - Timing analysis
  - State machine debugging
  - Bus transactions

##### Saleae Logic Analyzer Example
```
1. Connect channels to signals:
   - CH0: SPI CLK
   - CH1: SPI MOSI
   - CH2: SPI MISO
   - CH3: SPI CS
   
2. Configure sampling:
   - Sample rate: 24 MS/s
   - Duration: 1 second
   
3. Add analyzer:
   - Select SPI analyzer
   - Configure: CPOL=0, CPHA=0, MSB first
   
4. Capture and decode
   - View decoded transactions
   - Export to CSV
```

##### Protocol Decoders
- **UART**: Baud rate, parity, stop bits
- **I2C**: Address, read/write, ACK/NACK
- **SPI**: CPOL, CPHA, bit order
- **CAN**: Identifier, data, CRC
- **USB**: Packet type, endpoint, data

##### Timing Analysis
```
Example: I2C bus timing
- Measure setup time (tSU)
- Measure hold time (tHD)
- Verify clock stretching
- Check START/STOP conditions
- Verify against I2C spec (100 kHz, 400 kHz)
```

### Profiling Tools

#### Types of Profiling
- **CPU Profiling**: Where time is spent (hotspots)
- **Memory Profiling**: Heap usage, leaks
- **Cache Profiling**: Hit/miss rates
- **I/O Profiling**: Peripheral usage

#### gprof (GNU Profiler)

##### Usage
```bash
# Compile with profiling
arm-none-eabi-gcc -pg -g main.c -o program

# Run program (generates gmon.out)
./program

# Analyze with gprof
gprof program gmon.out > analysis.txt
```

##### Output
```
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total
 time   seconds   seconds    calls  Ts/call  Ts/call  name
 60.00      0.03     0.03    10000     0.00     0.00  sensor_process
 20.00      0.04     0.01     5000     0.00     0.00  filter_data
 10.00      0.05     0.01        1     0.01     0.05  main
 10.00      0.05     0.01     1000     0.00     0.00  calculate_average
```

#### perf (Linux Performance Analyzer)

##### Basic Commands
```bash
# Record performance data
sudo perf record -g ./program

# View report
sudo perf report

# Live monitoring
sudo perf top

# Specific events
sudo perf stat -e cache-misses,cache-references ./program
```

##### Flame Graphs
```bash
# Generate flame graph
perf record -F 99 -g ./program
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

#### Valgrind

##### Memory Leak Detection
```bash
valgrind --leak-check=full ./program

# Example output:
==12345== HEAP SUMMARY:
==12345==     in use at exit: 1,024 bytes in 1 blocks
==12345==   total heap usage: 5 allocs, 4 frees, 2,048 bytes allocated
==12345== 
==12345== 1,024 bytes in 1 blocks are definitely lost
==12345==    at 0x4C2AB80: malloc (in /usr/lib/valgrind/vgpreload_memcheck.so)
==12345==    by 0x4005B4: init_buffer (main.c:23)
==12345==    by 0x400612: main (main.c:45)
```

##### Cache Profiling
```bash
valgrind --tool=cachegrind ./program

# View results
cg_annotate cachegrind.out.<pid>
```

#### Embedded Profilers

##### SEGGER SystemView
- **Real-Time Trace**: RTOS tasks, interrupts, API calls
- **Zero-Cost**: Uses ARM trace (ITM/ETM)
- **Visualization**: Timeline, task states, CPU load

```c
// SystemView instrumentation
#include "SEGGER_SYSVIEW.h"

void task_function(void) {
    SEGGER_SYSVIEW_OnTaskStartExec(task_id);
    
    // Task work
    
    SEGGER_SYSVIEW_OnTaskStopExec();
}
```

##### Percepio Tracealyzer
- **FreeRTOS**: Integrated with FreeRTOS
- **Analysis**: Task scheduling, CPU usage, timing
- **Streaming**: Live trace capture

#### Manual Profiling with GPIO
```c
// Toggle GPIO to measure execution time
#define PROFILE_START()  GPIO_SET(PROFILE_PIN)
#define PROFILE_END()    GPIO_CLEAR(PROFILE_PIN)

void fast_algorithm(void) {
    PROFILE_START();
    
    // Algorithm code
    
    PROFILE_END();
}

// Measure pulse width with oscilloscope
```

### Memory Profiling

#### Heap Analysis
```c
// Track heap usage
typedef struct {
    size_t total_allocated;
    size_t current_usage;
    size_t peak_usage;
    size_t allocation_count;
    size_t free_count;
} HeapStats_t;

void *malloc_tracked(size_t size) {
    void *ptr = malloc(size);
    if (ptr) {
        heap_stats.total_allocated += size;
        heap_stats.current_usage += size;
        if (heap_stats.current_usage > heap_stats.peak_usage) {
            heap_stats.peak_usage = heap_stats.current_usage;
        }
        heap_stats.allocation_count++;
    }
    return ptr;
}

void free_tracked(void *ptr) {
    if (ptr) {
        // Would need to track sizes to update current_usage
        heap_stats.free_count++;
        free(ptr);
    }
}
```

#### Stack Usage Analysis
```c
// Fill stack with pattern
#define STACK_CANARY 0xA5A5A5A5

void fill_stack(uint32_t *stack_bottom, size_t stack_size) {
    for (size_t i = 0; i < stack_size / sizeof(uint32_t); i++) {
        stack_bottom[i] = STACK_CANARY;
    }
}

// Check stack usage
size_t get_stack_usage(uint32_t *stack_bottom, size_t stack_size) {
    size_t unused = 0;
    for (size_t i = 0; i < stack_size / sizeof(uint32_t); i++) {
        if (stack_bottom[i] != STACK_CANARY) {
            break;
        }
        unused += sizeof(uint32_t);
    }
    return stack_size - unused;
}
```

#### Memory Map Analysis
```bash
# View memory sections
arm-none-eabi-size -A firmware.elf

# Output:
firmware.elf  :
section              size      addr
.text              65536   0x8000000
.rodata             8192   0x8010000
.data               2048   0x20000000
.bss               16384   0x20000800
Total              92160

# Generate map file (shows all symbols)
arm-none-eabi-gcc -Wl,-Map=output.map ...

# Analyze with tools
python analyze_map.py output.map
```

### System Debugging Techniques

#### Printf Debugging (ITM)
```c
// Redirect printf to ITM
int _write(int file, char *ptr, int len) {
    for (int i = 0; i < len; i++) {
        ITM_SendChar(*ptr++);
    }
    return len;
}

// Use printf normally
printf("Temperature: %.2f\n", temp);
```

#### Assertion and Error Tracking
```c
// Enhanced assertions
#define ASSERT(expr) \
    do { \
        if (!(expr)) { \
            printf("Assertion failed: %s:%d: %s\n", \
                   __FILE__, __LINE__, #expr); \
            __BKPT(0); \
        } \
    } while(0)

// Error tracking
typedef struct {
    uint32_t error_code;
    const char *file;
    uint32_t line;
    uint32_t timestamp;
} ErrorLog_t;

ErrorLog_t error_log[16];
uint8_t error_index = 0;

void log_error(uint32_t code, const char *file, uint32_t line) {
    error_log[error_index].error_code = code;
    error_log[error_index].file = file;
    error_log[error_index].line = line;
    error_log[error_index].timestamp = get_timestamp();
    error_index = (error_index + 1) % 16;
}
```

#### Watchpoints
```gdb
# GDB watchpoint (break when variable changes)
(gdb) watch my_variable

# Break when variable equals value
(gdb) watch my_variable == 42

# Break on memory access
(gdb) watch *(uint32_t *)0x20000100
```

#### Core Dumps (Embedded)
```c
// Store CPU state on fault
typedef struct {
    uint32_t r0, r1, r2, r3;
    uint32_t r12, lr, pc, psr;
} CoreDump_t;

CoreDump_t core_dump __attribute__((section(".noinit")));

void HardFault_Handler(void) {
    // Save registers from stack
    uint32_t *stack;
    __asm volatile ("MRS %0, MSP" : "=r" (stack));
    
    core_dump.r0 = stack[0];
    core_dump.r1 = stack[1];
    core_dump.r2 = stack[2];
    core_dump.r3 = stack[3];
    core_dump.r12 = stack[4];
    core_dump.lr = stack[5];
    core_dump.pc = stack[6];
    core_dump.psr = stack[7];
    
    // Reboot
    NVIC_SystemReset();
}

// After reboot, analyze core_dump
```

## Key Concepts

- **Hotspot**: Code section consuming most CPU time
- **Memory Leak**: Allocated memory never freed
- **Cache Miss**: Data not in cache (slow access)
- **Stack Overflow**: Stack grows beyond allocated space

## Practical Exercises

1. Use oscilloscope to verify PWM signal
2. Debug I2C communication with logic analyzer
3. Profile application with gprof
4. Find memory leak with Valgrind
5. Analyze task scheduling with SystemView
6. Measure function execution time with GPIO
7. Create custom profiling system

## Best Practices

1. **Profile before optimizing** (measure, don't guess)
2. **Focus on hotspots** (80/20 rule)
3. **Use appropriate tool** (oscilloscope vs logic analyzer)
4. **Verify timing** with measurement equipment
5. **Track memory usage** during development
6. **Automate profiling** in CI/CD

## Tools Summary

| Tool | Purpose | Target |
|------|---------|--------|
| Oscilloscope | Analog signals | Hardware |
| Logic Analyzer | Digital signals | Hardware/Protocol |
| gprof | CPU profiling | Desktop/Linux |
| perf | Performance analysis | Linux |
| Valgrind | Memory debugging | Desktop/Linux |
| SystemView | RTOS trace | Embedded RTOS |
| GDB | Debugging | All |

## Resources

- Oscilloscope tutorial videos
- Saleae Logic software
- Valgrind documentation
- SEGGER SystemView user guide
- Linux perf wiki
