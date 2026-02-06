# 1.2 Low-Level Programming (C/C++ & Assembly)

## Topics Covered

### C Language (Expert)
- **Pointers and Pointer Arithmetic**
  - Pointer types and dereferencing
  - Function pointers and callbacks
  - Pointers to pointers and multi-level indirection
  - Const correctness and pointer qualifiers

- **Volatile Keyword**
  - When and why to use volatile
  - Hardware register access
  - Memory-mapped I/O
  - Compiler optimization barriers

- **Bit Manipulation**
  - Bitwise operators (AND, OR, XOR, NOT, shifts)
  - Bit masking and extraction
  - Setting, clearing, and toggling bits
  - Bit fields and packed structures

- **Struct Packing and Alignment**
  - Structure padding and alignment rules
  - Packed structures (#pragma pack)
  - Alignment requirements for different architectures
  - Union tricks and type punning

### Modern C++ for Embedded
- **C++11/14/17/20 Features**
  - Auto keyword and type inference
  - Range-based for loops
  - Lambda expressions
  - Move semantics
  - Constexpr for compile-time computation

- **RAII (Resource Acquisition Is Initialization)**
  - Automatic resource management
  - Scope-based cleanup
  - Exception safety

- **Smart Pointers**
  - unique_ptr for exclusive ownership
  - shared_ptr for shared ownership
  - weak_ptr to break circular references
  - Custom deleters for hardware resources

- **Templates**
  - Function templates
  - Class templates
  - Template specialization
  - Variadic templates
  - SFINAE and type traits

### Assembly Programming
- **Startup Code**
  - Vector table initialization
  - Stack pointer setup
  - BSS section zeroing
  - Data section initialization
  - Jump to main()

- **Inline Assembly**
  - GCC inline assembly syntax
  - Input/output operands
  - Clobber list
  - Memory barriers
  - Use cases: atomic operations, special instructions

- **Context Switching Code**
  - Saving processor state
  - Restoring processor state
  - Stack frame manipulation
  - RTOS task switching implementation

### Build Process
- **Preprocessor**
  - Macros and conditional compilation
  - Include guards and header files
  - Predefined macros
  - Macro best practices and pitfalls

- **Compiler**
  - Compilation stages
  - Optimization levels (-O0, -O1, -O2, -O3, -Os)
  - Warning flags and error checking
  - Cross-compilation

- **Linker**
  - Object file linking
  - Symbol resolution
  - Static vs dynamic linking
  - Link-time optimization (LTO)

- **Linker Scripts (.ld)**
  - Memory region definition
  - Section placement
  - Symbol definition and assignment
  - KEEP directive for startup code
  - Custom section attributes

- **Locator**
  - Address assignment
  - Memory map generation
  - Section alignment

## Key Concepts

- **Memory-mapped I/O vs Port-mapped I/O**
- **Endianness (Big-endian vs Little-endian)**
- **Calling conventions (AAPCS for ARM)**
- **Zero-cost abstractions in C++**

## Practical Exercises

1. Write memory-mapped register access macros
2. Implement a circular buffer using bit manipulation
3. Create a C++ peripheral wrapper class using RAII
4. Analyze and modify linker scripts
5. Write inline assembly for atomic operations
6. Implement a template-based hardware abstraction

## Code Examples

### Volatile Register Access
```c
#define GPIO_BASE 0x40020000
#define GPIOA_ODR (*(volatile uint32_t *)(GPIO_BASE + 0x14))

// Set bit 5
GPIOA_ODR |= (1 << 5);
```

### Bit Manipulation Macros
```c
#define SET_BIT(REG, BIT)     ((REG) |= (1U << (BIT)))
#define CLEAR_BIT(REG, BIT)   ((REG) &= ~(1U << (BIT)))
#define TOGGLE_BIT(REG, BIT)  ((REG) ^= (1U << (BIT)))
#define READ_BIT(REG, BIT)    (((REG) >> (BIT)) & 1U)
```

### C++ RAII for GPIO
```cpp
class GPIO_Pin {
    volatile uint32_t* port;
    uint8_t pin;
public:
    GPIO_Pin(volatile uint32_t* p, uint8_t n) : port(p), pin(n) {
        // Initialize pin
    }
    ~GPIO_Pin() {
        // Cleanup if needed
    }
    void set() { *port |= (1 << pin); }
    void clear() { *port &= ~(1 << pin); }
};
```

## References

- C11/C17 Standard
- C++11/14/17/20 Standards
- ARM Assembly Language Fundamentals
- GCC documentation
- Linker script tutorials
