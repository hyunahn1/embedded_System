# 2.1 RTOS Kernel Internals

## Topics Covered

### Task Scheduling Algorithms

#### Preemptive Scheduling
- **Priority-Based Preemptive Scheduling**
  - Fixed-priority scheduling
  - Task priorities and precedence
  - Preemption points
  - Priority levels configuration
  - Rate Monotonic Scheduling (RMS)

- **Preemption Mechanisms**
  - Voluntary preemption (yield)
  - Involuntary preemption (higher priority task ready)
  - Interrupt-driven preemption
  - Critical sections and preemption disabling

#### Round-Robin Scheduling
- Time-slice (quantum) allocation
- Task rotation at equal priority
- Combining priority-based and round-robin
- Configuring time-slice duration
- Fairness considerations

#### Cooperative Scheduling
- Task yielding
- Non-preemptive execution
- When to use cooperative scheduling
- Advantages and disadvantages
- Hybrid scheduling models

#### Advanced Scheduling
- Earliest Deadline First (EDF)
- Least Laxity First (LLF)
- Rate Monotonic Analysis
- Utilization bounds
- Schedulability analysis

### Context Switching Mechanics

#### Saving/Restoring State
- **Register Saving**
  - General-purpose registers
  - Stack pointer (PSP in ARM)
  - Program counter
  - Status registers
  - FPU registers (if applicable)

- **Context Switch Sequence**
  1. Save current task context
  2. Select next task to run
  3. Restore next task context
  4. Resume execution

- **Hardware Support**
  - ARM PendSV exception for context switching
  - Tail-chaining optimization
  - Lazy context save/restore (FPU)

- **Software Implementation**
  - Inline assembly for context save/restore
  - Stack frame structure
  - Task Control Block (TCB)

#### Context Switch Overhead
- Measurement techniques
- Optimization strategies
- Cache effects
- Interrupt latency impact

### Stack Overflow Detection & Analysis

#### Stack Basics
- Stack growth direction
- Stack allocation per task
- Stack pointer (SP) management
- Stack frame anatomy

#### Stack Size Determination
- Static analysis
- Runtime profiling
- Worst-case estimation
- Stack watermarking

#### Overflow Detection Methods
- **Software Methods**
  - Stack canaries (magic numbers)
  - Stack watermarking
  - Periodic stack checking
  - Guard pages

- **Hardware Methods**
  - MPU-based stack protection
  - Stack limit registers
  - Hardware watchpoints

#### Debugging Stack Issues
- Analyzing stack dumps
- Backtrace generation
- Identifying overflow sources
- Tools and techniques

### Tickless Idle & Power Management

#### System Tick
- Periodic timer interrupt
- Tick rate configuration (1ms, 10ms typical)
- Time slicing implementation
- Tick count maintenance

#### Tickless Idle Mode
- **Concept**
  - Disable periodic tick during idle
  - Wake on next scheduled event
  - Dynamic tick suppression

- **Implementation**
  - Calculate sleep duration
  - Configure low-power timer
  - Handle early wakeup
  - Compensate tick count

- **Benefits**
  - Reduced power consumption
  - Lower interrupt overhead
  - Extended battery life

#### Power Management
- **CPU Sleep Modes**
  - Sleep mode
  - Deep sleep mode
  - Standby/shutdown modes
  - Mode selection criteria

- **RTOS Integration**
  - Idle task hook
  - Pre-sleep and post-sleep processing
  - Peripheral clock gating
  - Voltage scaling

- **Tickless + Power Modes**
  - Coordinated sleep entry
  - Wakeup source configuration
  - Timing accuracy considerations

## Key Concepts

- **Task States**: Ready, Running, Blocked, Suspended
- **Task Priority**: Higher number = higher priority (typically)
- **Task Control Block (TCB)**: Kernel data structure per task
- **Idle Task**: Lowest priority task, runs when nothing else can
- **Kernel Objects**: Tasks, Queues, Semaphores, Mutexes, etc.

## Practical Exercises

1. Implement a simple round-robin scheduler
2. Write context switching code in assembly
3. Add stack overflow detection to tasks
4. Measure context switch overhead
5. Implement tickless idle mode
6. Create a task profiling system
7. Analyze schedulability of a task set

## Code Examples

### Task Control Block Structure
```c
typedef struct {
    void *stack_pointer;      // Current stack pointer
    uint32_t *stack_base;     // Bottom of stack
    uint32_t stack_size;      // Stack size in bytes
    uint8_t priority;         // Task priority
    uint8_t state;            // Task state
    char name[16];            // Task name
    // ... other fields
} TCB_t;
```

### Context Switch (ARM Cortex-M)
```c
__attribute__((naked)) void PendSV_Handler(void) {
    __asm volatile (
        "mrs r0, psp                \n" // Get current PSP
        "stmdb r0!, {r4-r11}        \n" // Save r4-r11
        "ldr r1, =current_tcb       \n"
        "ldr r2, [r1]               \n"
        "str r0, [r2]               \n" // Save PSP to TCB
        
        // Select next task
        "bl scheduler_select_next   \n"
        
        "ldr r1, =current_tcb       \n"
        "ldr r2, [r1]               \n"
        "ldr r0, [r2]               \n" // Load PSP from TCB
        "ldmia r0!, {r4-r11}        \n" // Restore r4-r11
        "msr psp, r0                \n" // Set PSP
        "bx lr                      \n" // Return
    );
}
```

### Stack Overflow Detection
```c
#define STACK_CANARY 0xDEADBEEF

void task_create(TCB_t *tcb, void *stack, uint32_t size) {
    // Place canary at bottom of stack
    uint32_t *canary = (uint32_t *)stack;
    *canary = STACK_CANARY;
    
    tcb->stack_base = stack;
    tcb->stack_size = size;
    // ... initialize stack for task
}

void check_stack_overflow(TCB_t *tcb) {
    uint32_t *canary = (uint32_t *)tcb->stack_base;
    if (*canary != STACK_CANARY) {
        // Stack overflow detected!
        error_handler("Stack overflow in task");
    }
}
```

### Tickless Idle Implementation
```c
void vPortSuppressTicksAndSleep(TickType_t xExpectedIdleTime) {
    uint32_t sleep_ticks;
    
    // Enter critical section
    __disable_irq();
    
    // Check if we can still sleep
    if (eTaskConfirmSleepModeStatus() == eAbortSleep) {
        __enable_irq();
        return;
    }
    
    // Calculate sleep duration
    sleep_ticks = xExpectedIdleTime - 1;
    
    // Configure low-power timer
    configure_lptim(sleep_ticks);
    
    // Enter sleep mode
    __WFI();
    
    // Compensate tick count
    vTaskStepTick(sleep_ticks);
    
    __enable_irq();
}
```

## Performance Metrics

- **Context switch time**: 1-5 microseconds (typical)
- **Interrupt latency**: <10 instruction cycles (with proper NVIC config)
- **Kernel overhead**: 2-5% CPU utilization
- **Memory footprint**: 2-10KB (kernel only)

## References

- FreeRTOS Kernel Developer Docs
- ARM Cortex-M Programming Guide
- "Real-Time Systems" by Jane W. S. Liu
- RTOS vendor documentation
