# 2.2 Synchronization & IPC (Inter-Process Communication)

## Topics Covered

### Mutex vs Semaphore

#### Mutex (Mutual Exclusion)
- **Purpose**: Protect shared resources from concurrent access
- **Ownership**: Has ownership concept (only owner can release)
- **Recursion**: Recursive mutex allows same task to lock multiple times
- **Use Cases**: 
  - Protecting shared variables
  - Guarding hardware peripherals
  - Serializing access to non-reentrant functions

- **Implementation**
  - Lock/unlock operations
  - Blocking vs non-blocking acquisition
  - Timeout mechanisms
  - Error handling

#### Semaphore
- **Binary Semaphore**
  - 0 or 1 value
  - Similar to mutex but no ownership
  - Signaling between tasks
  - ISR to task synchronization

- **Counting Semaphore**
  - Counts available resources
  - Managing resource pools
  - Rate limiting
  - Implementing barriers

- **Key Differences from Mutex**
  - No ownership requirement
  - Can be given from ISR
  - Can be used for synchronization (not just mutual exclusion)

### Priority Inversion & Inheritance

#### Priority Inversion Problem
- **Scenario**
  - High-priority task blocked by low-priority task
  - Medium-priority task preempts low-priority task
  - High-priority task effectively delayed by medium-priority task

- **Unbounded Priority Inversion**
  - Mars Pathfinder incident
  - System deadlock scenarios
  - Real-world consequences

#### Priority Inheritance Protocol
- **Mechanism**
  - Low-priority task inherits priority of highest waiting task
  - Temporary priority boost
  - Priority restored after resource release

- **Implementation**
  - Kernel tracks mutex ownership
  - Dynamic priority adjustment
  - Nested inheritance handling

#### Priority Ceiling Protocol
- **Mechanism**
  - Mutex has ceiling priority
  - Task inherits ceiling when acquiring mutex
  - Prevents medium-priority preemption

- **Advantages**
  - Bounded blocking time
  - Deadlock prevention (with proper ceiling assignment)

### Message Queues & Mailboxes

#### Message Queues
- **Characteristics**
  - FIFO data structure
  - Fixed-size messages
  - Multiple producers/consumers
  - Blocking on empty/full conditions

- **Operations**
  - Send (post) message
  - Receive (pend) message
  - Peek without removing
  - Timeout options

- **Use Cases**
  - Inter-task communication
  - Data buffering
  - Event logging
  - Command dispatching

- **Implementation Details**
  - Ring buffer structure
  - Message copying vs reference
  - Queue size determination
  - Memory allocation (static vs dynamic)

#### Mailboxes
- **Characteristics**
  - Single message slot
  - Overwrite vs blocking semantics
  - Lightweight alternative to queues

- **Typical Usage**
  - Status updates
  - Latest value semantics
  - Simple synchronization

### Event Flags & Signal Groups

#### Event Flags
- **Concept**
  - Bit-field of binary events
  - Each bit represents an event
  - Tasks wait for event combinations

- **Operations**
  - Set flag(s)
  - Clear flag(s)
  - Wait for ANY flag (OR condition)
  - Wait for ALL flags (AND condition)

- **Use Cases**
  - Multiple event synchronization
  - State machine triggers
  - Complex task coordination

#### Signal Groups
- **Similar to Event Flags**
  - Task-specific signals
  - Group-based notifications
  - Broadcast signaling

### Deadlock & Race Condition Prevention

#### Deadlock
- **Necessary Conditions (Coffman Conditions)**
  1. Mutual exclusion
  2. Hold and wait
  3. No preemption
  4. Circular wait

- **Prevention Strategies**
  - Lock ordering protocol
  - Timeout on lock acquisition
  - Resource allocation graphs
  - Avoiding nested locks

- **Detection and Recovery**
  - Watchdog timers
  - Deadlock detection algorithms
  - System reset as last resort

#### Race Conditions
- **Definition**: Output depends on sequence/timing of events

- **Common Scenarios**
  - Non-atomic read-modify-write
  - Unprotected shared variables
  - Incorrect synchronization

- **Prevention**
  - Critical sections
  - Atomic operations
  - Proper mutex usage
  - Volatile keyword (where appropriate)

### Critical Sections

#### Disabling Interrupts
- **Method**: `__disable_irq()` / `__enable_irq()`
- **Pros**: 
  - Simplest implementation
  - Guaranteed atomicity
- **Cons**: 
  - Affects all interrupts
  - Increases interrupt latency
  - Should be very short (<10-20 instructions)

- **Use Cases**
  - Very short critical sections
  - Interrupt service routines
  - System-level operations

#### Spinlocks
- **Concept**: Busy-wait for lock availability
- **Implementation**: Atomic test-and-set operations
- **Use in Multi-core**: Synchronization between cores
- **Pros**: Low overhead for short waits
- **Cons**: Wastes CPU cycles, not suitable for long waits

#### RTOS Critical Sections
- **Suspend Scheduler**: `vTaskSuspendAll()` / `xTaskResumeAll()`
- **Allows**: Interrupts still serviced
- **Prevents**: Context switches
- **Use Cases**: 
  - Protecting data from other tasks (not ISRs)
  - Multiple kernel API calls atomically

## Key Concepts

- **Synchronization**: Coordinating task execution
- **Communication**: Exchanging data between tasks
- **Mutual Exclusion**: Preventing concurrent access
- **Determinism**: Predictable worst-case behavior
- **Bounded Waiting**: Maximum wait time is finite

## Design Patterns

### Producer-Consumer
```
Producer Task -> Queue -> Consumer Task
```

### Rendezvous Pattern
```
Task A ---|
          +--> Synchronization Point --> Continue
Task B ---|
```

### Barrier Pattern
```
All tasks must reach barrier before any proceed
```

## Practical Exercises

1. Implement a mutex-protected shared resource
2. Demonstrate priority inversion and fix with priority inheritance
3. Create producer-consumer system with queue
4. Use event flags for multi-condition synchronization
5. Detect and resolve a deadlock scenario
6. Implement a rate-limiting semaphore
7. Create a mailbox for inter-task status updates

## Code Examples

### Mutex Usage
```c
SemaphoreHandle_t xMutex;

void vTask1(void *pvParameters) {
    xMutex = xSemaphoreCreateMutex();
    
    for (;;) {
        if (xSemaphoreTake(xMutex, portMAX_DELAY) == pdTRUE) {
            // Critical section - access shared resource
            shared_resource++;
            
            xSemaphoreGive(xMutex);
        }
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

### Message Queue
```c
QueueHandle_t xQueue;

typedef struct {
    uint8_t id;
    uint32_t value;
} Message_t;

// Producer
void vProducerTask(void *pvParameters) {
    xQueue = xQueueCreate(10, sizeof(Message_t));
    
    for (;;) {
        Message_t msg = {.id = 1, .value = get_sensor_data()};
        xQueueSend(xQueue, &msg, portMAX_DELAY);
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}

// Consumer
void vConsumerTask(void *pvParameters) {
    Message_t msg;
    
    for (;;) {
        if (xQueueReceive(xQueue, &msg, portMAX_DELAY) == pdTRUE) {
            process_message(&msg);
        }
    }
}
```

### Event Flags
```c
EventGroupHandle_t xEventGroup;

#define BIT_0 (1 << 0)
#define BIT_1 (1 << 1)
#define BIT_2 (1 << 2)

void vTask1(void *pvParameters) {
    xEventGroup = xEventGroupCreate();
    
    // Wait for BIT_0 AND BIT_1
    EventBits_t uxBits = xEventGroupWaitBits(
        xEventGroup,
        BIT_0 | BIT_1,
        pdTRUE,  // Clear on exit
        pdTRUE,  // Wait for all bits
        portMAX_DELAY
    );
    
    // Both events occurred
    handle_combined_event();
}

void vTask2(void *pvParameters) {
    // Set BIT_0
    xEventGroupSetBits(xEventGroup, BIT_0);
}
```

### Avoiding Deadlock with Lock Ordering
```c
SemaphoreHandle_t xMutex1, xMutex2;

// GOOD: Consistent lock ordering
void vTask_GoodOrder(void *pvParameters) {
    // Always acquire in order: Mutex1 then Mutex2
    xSemaphoreTake(xMutex1, portMAX_DELAY);
    xSemaphoreTake(xMutex2, portMAX_DELAY);
    
    // Critical section
    
    xSemaphoreGive(xMutex2);
    xSemaphoreGive(xMutex1);
}

// BAD: Inconsistent lock ordering can cause deadlock
void vTask_BadOrder(void *pvParameters) {
    // Acquires in opposite order - DEADLOCK RISK
    xSemaphoreTake(xMutex2, portMAX_DELAY);
    xSemaphoreTake(xMutex1, portMAX_DELAY);
    
    // Critical section
    
    xSemaphoreGive(xMutex1);
    xSemaphoreGive(xMutex2);
}
```

## Best Practices

1. **Keep critical sections short**
2. **Use mutexes for mutual exclusion, semaphores for synchronization**
3. **Always check return values**
4. **Use timeouts to prevent infinite blocking**
5. **Maintain consistent lock ordering**
6. **Avoid calling blocking functions from ISRs**
7. **Use priority inheritance mutexes by default**
8. **Document synchronization requirements**

## Common Pitfalls

- Forgetting to release mutex
- Taking mutex from ISR (not allowed)
- Deadlock due to circular dependencies
- Race conditions from insufficient synchronization
- Priority inversion without mitigation

## References

- FreeRTOS Synchronization documentation
- "Operating Systems: Three Easy Pieces" by Remzi Arpaci-Dusseau
- "Embedded Systems: Real-Time Operating Systems" by Jonathan Valvano
- RTOS vendor API references
