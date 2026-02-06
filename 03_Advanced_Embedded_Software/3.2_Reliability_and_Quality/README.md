# 3.2 Reliability & Quality Standards

## Topics Covered

### MISRA C/C++ Guidelines

#### Overview
- **MISRA**: Motor Industry Software Reliability Association
- **Purpose**: Improve safety, security, and reliability
- **Adoption**: Automotive, aerospace, medical, industrial
- **Versions**: MISRA C:2012, MISRA C++:2008, MISRA C:2023

#### Key Principles
- **Mandatory Rules**: Must be followed
- **Required Rules**: Should be followed (deviations documented)
- **Advisory Rules**: Recommended best practices

#### Categories
- **Decidable**: Can be automatically checked
- **Undecidable**: Require manual review

#### Common MISRA C Rules

**Rule 1.3**: No undefined behavior
- Avoid signed integer overflow
- No out-of-bounds array access
- Proper pointer usage

**Rule 8.4**: Compatible function declarations
```c
// Bad: Implicit int
func();

// Good: Explicit types
int func(void);
```

**Rule 10.1-10.8**: Type conversions
- Avoid implicit conversions
- Be explicit with casts
- No mixed-mode arithmetic

**Rule 14.3**: Controlling expressions should not be invariant
```c
// Bad: Condition always true
if (x || 1) { }

// Good: Meaningful condition
if (x > threshold) { }
```

**Rule 17.7**: Return values should be checked
```c
// Bad: Ignoring return value
scanf("%d", &value);

// Good: Checking return value
if (scanf("%d", &value) != 1) {
    // Handle error
}
```

**Rule 21.3**: malloc/free prohibited (in some contexts)
- Prefer static allocation
- Use memory pools if dynamic needed

#### MISRA Compliance
- **Static Analysis Tools**
  - PC-Lint Plus
  - Polyspace
  - Coverity
  - Cppcheck (with MISRA addon)

- **Deviation Process**
  - Document why rule is deviated
  - Risk assessment
  - Alternative measures
  - Approval process

### Defensive Programming

#### Principles
- **Distrust Input**: Validate all inputs
- **Fail Safe**: Design for graceful failure
- **Assertions**: Check assumptions
- **Error Handling**: Always handle errors

#### Input Validation
```c
// Validate range
int set_speed(int speed) {
    if (speed < SPEED_MIN || speed > SPEED_MAX) {
        return ERROR_INVALID_PARAM;
    }
    motor_speed = speed;
    return SUCCESS;
}

// Validate pointers
int process_data(const uint8_t *data, size_t len) {
    if (data == NULL || len == 0) {
        return ERROR_NULL_POINTER;
    }
    // Process data
    return SUCCESS;
}
```

#### Assertions
```c
#include <assert.h>

void configure_timer(uint32_t prescaler) {
    assert(prescaler > 0 && prescaler <= MAX_PRESCALER);
    // Configuration code
}

// Custom assert with error logging
#define ASSERT(condition) \
    do { \
        if (!(condition)) { \
            log_error("Assertion failed: " #condition); \
            error_handler(); \
        } \
    } while(0)
```

#### Error Codes
```c
typedef enum {
    SUCCESS = 0,
    ERROR_NULL_POINTER,
    ERROR_INVALID_PARAM,
    ERROR_TIMEOUT,
    ERROR_HARDWARE_FAULT,
    ERROR_OUT_OF_MEMORY
} ErrorCode_t;

// Always return error codes
ErrorCode_t init_device(void) {
    if (!hardware_ready()) {
        return ERROR_HARDWARE_FAULT;
    }
    // Initialization
    return SUCCESS;
}

// Check return values
ErrorCode_t result = init_device();
if (result != SUCCESS) {
    handle_error(result);
}
```

#### Resource Management
```c
// RAII-style pattern in C
typedef struct {
    FILE *file;
} FileHandle_t;

FileHandle_t *open_file(const char *path) {
    FileHandle_t *handle = malloc(sizeof(FileHandle_t));
    if (handle) {
        handle->file = fopen(path, "r");
        if (!handle->file) {
            free(handle);
            return NULL;
        }
    }
    return handle;
}

void close_file(FileHandle_t *handle) {
    if (handle) {
        if (handle->file) {
            fclose(handle->file);
        }
        free(handle);
    }
}
```

### Debugging Tools & Techniques

#### GDB (GNU Debugger)
- **Basic Commands**
  ```
  break main          # Set breakpoint
  run                 # Start program
  continue            # Continue execution
  step                # Step into
  next                # Step over
  print variable      # Print value
  backtrace           # Show call stack
  ```

- **Embedded GDB**
  - Connect via GDB server (OpenOCD, J-Link)
  - Load symbol file
  - Hardware breakpoints
  - Memory inspection

#### JTAG/SWD
- **JTAG**: Joint Test Action Group
  - 4 or 5 wire interface
  - Boundary scan, debugging
  
- **SWD**: Serial Wire Debug (ARM)
  - 2 wire interface (SWDIO, SWCLK)
  - More efficient than JTAG
  - Preferred for Cortex-M

- **Tools**
  - SEGGER J-Link
  - ST-Link
  - OpenOCD (open source)

#### Trace (ETM/ITM)
- **ETM**: Embedded Trace Macrocell
  - Non-intrusive instruction trace
  - Requires trace port
  - Full program flow reconstruction

- **ITM**: Instrumentation Trace Macrocell
  - Printf-style debugging
  - Low overhead
  - Timestamps
  - Available on Cortex-M3/M4/M7

- **SWO**: Single Wire Output
  - Serial trace data
  - ITM + DWT output

```c
// ITM printf example
void ITM_SendChar(char ch) {
    while (ITM->PORT[0].u32 == 0);
    ITM->PORT[0].u8 = ch;
}

void ITM_printf(const char *fmt, ...) {
    char buffer[256];
    va_list args;
    va_start(args, fmt);
    vsnprintf(buffer, sizeof(buffer), fmt, args);
    va_end(args);
    
    for (char *p = buffer; *p; p++) {
        ITM_SendChar(*p);
    }
}
```

#### Semihosting
- **Concept**: Use debugger as I/O channel
- **Operations**: File I/O, console output
- **Overhead**: Very slow, debug only
- **Usage**: Early bring-up, testing

```c
// printf redirected through semihosting
extern void initialise_monitor_handles(void);

int main(void) {
    initialise_monitor_handles();
    printf("Hello from target!\n");
}
```

#### Logic Analyzer
- **Hardware Tool**: Capture digital signals
- **Use Cases**:
  - Protocol debugging (I2C, SPI, UART)
  - Timing analysis
  - Signal integrity
- **Popular Tools**: Saleae Logic, DSLogic

#### Oscilloscope
- **Use Cases**:
  - Analog signal debugging
  - PWM verification
  - Power supply analysis
  - Timing measurements

### Testing Methodologies

#### Unit Testing
- **Frameworks**
  - **GoogleTest/GoogleMock**: C++ testing
  - **Unity**: C testing framework
  - **Ceedling**: Unity + CMock + Rake
  - **CppUTest**: C/C++ testing

- **Example with Unity**
  ```c
  #include "unity.h"
  #include "my_module.h"
  
  void setUp(void) {
      // Run before each test
      module_init();
  }
  
  void tearDown(void) {
      // Run after each test
      module_cleanup();
  }
  
  void test_function_should_return_correct_value(void) {
      int result = calculate(2, 3);
      TEST_ASSERT_EQUAL_INT(5, result);
  }
  
  int main(void) {
      UNITY_BEGIN();
      RUN_TEST(test_function_should_return_correct_value);
      return UNITY_END();
  }
  ```

#### Mocking Hardware
- **Problem**: Unit tests shouldn't depend on hardware
- **Solution**: Hardware Abstraction Layer (HAL) + Mocking

```c
// HAL interface
typedef struct {
    void (*gpio_set)(int pin);
    void (*gpio_clear)(int pin);
    int (*gpio_read)(int pin);
} GPIO_HAL_t;

// Real implementation
void gpio_set_real(int pin) {
    GPIOA->BSRR = (1 << pin);
}

// Mock implementation for testing
static int mock_gpio_state[16];
void gpio_set_mock(int pin) {
    mock_gpio_state[pin] = 1;
}

// Use function pointers or dependency injection
GPIO_HAL_t gpio_hal = {
    .gpio_set = gpio_set_real,  // Production
    // .gpio_set = gpio_set_mock,  // Testing
};
```

#### Integration Testing
- **Target**: Test module interactions
- **Hardware-in-the-Loop (HIL)**
  - Real hardware, automated tests
  - Simulated inputs/outputs
  - Continuous integration

#### Coverage Analysis
- **Types**:
  - Statement coverage
  - Branch coverage
  - Function coverage
  - Path coverage

- **Tools**:
  - gcov/lcov
  - Bullseye Coverage
  - Embedded coverage tools

- **Target**: 70-80% is good, 100% is rarely practical

#### Static Analysis
- **Tools**:
  - Clang Static Analyzer
  - Cppcheck
  - Coverity
  - SonarQube

- **Checks**:
  - Buffer overflows
  - Null pointer dereferences
  - Memory leaks
  - Dead code
  - MISRA violations

#### Continuous Integration
- **Automated Builds**: Every commit
- **Automated Tests**: Unit, integration
- **Static Analysis**: On every build
- **Metrics**: Code coverage, complexity

## Key Concepts

- **Defense in Depth**: Multiple layers of protection
- **Fail Fast**: Detect errors as early as possible
- **Test-Driven Development (TDD)**: Write tests first
- **Code Reviews**: Peer review catches bugs

## Practical Exercises

1. Run static analysis on existing code
2. Write unit tests for a module
3. Mock hardware interfaces
4. Debug with GDB and breakpoints
5. Use ITM for printf debugging
6. Measure code coverage
7. Set up CI pipeline for embedded project

## Best Practices

1. **Follow coding standards** (MISRA, CERT C)
2. **Use static analysis tools** regularly
3. **Write testable code** (loose coupling, dependency injection)
4. **Validate all inputs**
5. **Check all return values**
6. **Use assertions liberally** (in debug builds)
7. **Automate testing**
8. **Peer review all code**

## Safety Standards

- **ISO 26262**: Automotive functional safety
- **IEC 61508**: Industrial functional safety
- **DO-178C**: Aviation software
- **IEC 62304**: Medical device software
- **Common criteria**: SIL levels (Safety Integrity Levels)

## References

- MISRA C:2012 Guidelines
- "Embedded C Coding Standard" by Michael Barr
- "Test Driven Development for Embedded C" by James Grenning
- Static analysis tool documentation
- Debugging tool user guides
