# 1.1 Processor & Architecture

## Topics Covered

### ARM Cortex-M Architecture (M0-M7)
- Cortex-M0/M0+: Ultra-low power, minimal instruction set
- Cortex-M3: Full Thumb-2 instruction set
- Cortex-M4: DSP extensions and floating-point unit (FPU)
- Cortex-M7: High-performance with cache and TCM
- Architecture comparison and selection criteria

### Registers (R0-R15, PC, SP, LR) & Flag Registers
- General-purpose registers (R0-R12)
- Stack Pointer (SP): MSP vs PSP
- Link Register (LR): Function calls and returns
- Program Counter (PC)
- Application Program Status Register (APSR)
- Special registers: PRIMASK, FAULTMASK, BASEPRI, CONTROL

### Interrupt Vector Table & NVIC
- Vector table structure and location
- Nested Vectored Interrupt Controller (NVIC)
- Interrupt priority levels and grouping
- Interrupt handling and latency
- Tail-chaining and late-arriving interrupts

### Memory Map (Flash, RAM, Peripherals)
- Standard Cortex-M memory map
- Flash memory organization (code storage)
- SRAM regions and usage
- Peripheral memory regions
- Bit-banding (Cortex-M3/M4)

### Stack vs Heap (Detailed Management)
- Stack operation and stack frames
- Stack overflow detection
- Heap allocation strategies
- Static vs dynamic memory allocation
- Memory pools and custom allocators

### Cache Coherency & MMU/MPU
- Data cache and instruction cache
- Cache coherency issues and solutions
- Memory Management Unit (MMU) - Cortex-A series
- Memory Protection Unit (MPU) - Cortex-M series
- MPU region configuration and access permissions

### Bootloader Basics (General)
- Boot process and startup code
- Bootloader responsibilities
- Application jump and vector table relocation
- Firmware update mechanisms
- Secure boot concepts

## Key Concepts

- **Von Neumann vs Harvard Architecture**
- **Pipeline stages and instruction execution**
- **Exception and fault handling**
- **Power modes and clock management**

## Practical Exercises

1. Analyze startup assembly code
2. Configure NVIC priorities
3. Implement MPU protection
4. Write a simple bootloader
5. Optimize stack and heap usage

## References

- ARM Cortex-M Programming Guide
- ARM Architecture Reference Manual
- Vendor-specific MCU reference manuals
