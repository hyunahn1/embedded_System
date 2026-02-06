# 2.3 RTOS Ecosystem

## Topics Covered

### FreeRTOS (Deep Dive)

#### Overview
- **History**: Created by Richard Barry, now maintained by AWS
- **License**: MIT (Open Source)
- **Popularity**: Most widely used RTOS
- **Support**: Wide MCU support, extensive community

#### Architecture
- **Kernel Components**
  - Task management
  - Queue management
  - Semaphore/Mutex management
  - Event groups
  - Timers
  - Memory management

- **Memory Management Schemes**
  - heap_1: Simplest, allocation only
  - heap_2: Best-fit, no coalescing
  - heap_3: Thread-safe malloc/free wrapper
  - heap_4: Best-fit with coalescing
  - heap_5: Multiple non-contiguous heaps

#### Configuration (FreeRTOSConfig.h)
- **Critical Settings**
  ```c
  #define configUSE_PREEMPTION                    1
  #define configUSE_TICKLESS_IDLE                 0
  #define configCPU_CLOCK_HZ                      SystemCoreClock
  #define configTICK_RATE_HZ                      1000
  #define configMAX_PRIORITIES                    5
  #define configMINIMAL_STACK_SIZE                128
  #define configTOTAL_HEAP_SIZE                   10240
  #define configMAX_TASK_NAME_LEN                 16
  #define configUSE_16_BIT_TICKS                  0
  #define configIDLE_SHOULD_YIELD                 1
  ```

- **Feature Enables**
  - Mutexes, semaphores, queues
  - Timers, event groups
  - Co-routines (rarely used)
  - Task notifications (lightweight)

#### API Categories
- **Task Control**: Create, delete, suspend, resume, delay
- **Queue Management**: Send, receive, peek
- **Semaphore/Mutex**: Take, give
- **Event Groups**: Set, clear, wait
- **Timers**: Software timers (daemon task)
- **Direct-to-Task Notifications**: Lightweight alternative to queues/semaphores

#### Port-Specific Code
- **ARM Cortex-M**: Uses PendSV for context switching
- **Port Files**: port.c, portmacro.h
- **Interrupt Management**: portYIELD_FROM_ISR()

#### AWS IoT Integration
- FreeRTOS+ libraries
- MQTT, HTTP, OTA updates
- Security features (TLS)

#### Best Practices
- Use task notifications when possible (faster, less memory)
- Choose appropriate heap scheme
- Monitor stack usage (uxTaskGetStackHighWaterMark)
- Use trace tools (Tracealyzer, SystemView)

### Zephyr OS

#### Overview
- **Maintainer**: Linux Foundation
- **License**: Apache 2.0
- **Philosophy**: Modular, scalable, security-focused
- **Support**: ARM, x86, RISC-V, Xtensa, ARC

#### Architecture
- **Kernel**: Microkernel architecture
- **Device Model**: Unified device tree
- **Build System**: CMake + Kconfig
- **Package Management**: West tool

#### Key Features
- **Native Linux Development**: Better tooling integration
- **Memory Protection**: User/kernel space separation (on supported HW)
- **Security**: ASLR, stack protection, MPU/MMU support
- **Networking**: Full TCP/IP stack (IPv4/IPv6)
- **Bluetooth**: Comprehensive BLE stack
- **Device Drivers**: Extensive driver library

#### Device Tree
- Hardware description in DTS format
- Compile-time configuration
- Overlays for board variants

#### Kconfig
- Feature selection
- Hierarchical configuration
- Automatic dependency resolution

#### Threads & Scheduling
- Multiple scheduling policies
- Cooperative and preemptive threads
- Work queues
- Polling API

#### Synchronization
- Mutexes, semaphores
- Condition variables
- FIFOs, LIFOs
- Pipes

#### Networking
- BSD sockets API
- CoAP, MQTT, LwM2M
- HTTP client/server
- Shell commands

#### Best Practices
- Use device tree for hardware abstraction
- Leverage Kconfig for feature selection
- Follow coding guidelines (Zephyr style)
- Use West for project management

### ThreadX

#### Overview
- **Vendor**: Microsoft (acquired from Express Logic)
- **License**: MIT (as of 2019)
- **Certification**: Safety-certified (IEC 61508, IEC 62304)
- **Part of**: Azure RTOS family

#### Architecture
- **Kernel**: ThreadX RTOS
- **FileSystem**: FileX (FAT)
- **Networking**: NetX Duo (IPv4/IPv6)
- **USB**: USBX
- **GUI**: GUIX

#### Key Features
- **Event Chaining**: Automatic event propagation
- **Picokernel**: Extremely small footprint
- **Performance**: Very low overhead
- **Priority Inheritance**: Built-in
- **Preemption Threshold**: Unique scheduling feature

#### Preemption Threshold Scheduling
- Combines cooperative and preemptive benefits
- Task can disable preemption by higher priorities up to threshold
- Reduces context switches, improves determinism

#### API Design
- Consistent naming (tx_*)
- Simple, intuitive calls
- No dynamic memory in core services

#### Safety Certification
- Pre-certified for medical (FDA)
- Automotive (ISO 26262)
- Industrial (IEC 61508)
- DO-178B (aviation)

#### Azure RTOS Integration
- IoT device connectivity
- OTA updates
- Security features
- Cloud integration

#### Best Practices
- Use preemption threshold strategically
- Leverage certified components for safety-critical apps
- Take advantage of FileX/NetX integration

## Comparison Matrix

| Feature | FreeRTOS | Zephyr | ThreadX |
|---------|----------|--------|---------|
| **License** | MIT | Apache 2.0 | MIT |
| **Footprint** | Very Small | Medium | Very Small |
| **Networking** | Limited (lwIP) | Full Stack | NetX Duo |
| **Safety Cert** | No (but can be certified) | No | Yes |
| **Community** | Very Large | Growing | Smaller |
| **Learning Curve** | Low | Medium | Low |
| **IoT Focus** | AWS IoT | Linux IoT | Azure IoT |
| **Memory Protection** | No (SW only) | Yes (with HW) | Yes |

## Selecting an RTOS

### Choose FreeRTOS if:
- Simple application with basic RTOS needs
- Large community support is important
- AWS IoT integration desired
- Want minimal learning curve

### Choose Zephyr if:
- Need modern development environment
- Want comprehensive networking/Bluetooth
- Require memory protection
- Prefer Linux-style development

### Choose ThreadX if:
- Need safety certification
- Want lowest possible overhead
- Azure IoT integration desired
- Preemption threshold scheduling beneficial

## Migration Considerations
- API differences
- Configuration approach
- Build system integration
- Toolchain compatibility
- Driver availability

## Practical Exercises

1. Create "Hello World" task in all three RTOS
2. Implement producer-consumer with queues in each
3. Compare memory footprint for identical application
4. Measure context switch overhead
5. Port an application from one RTOS to another
6. Use device tree in Zephyr for GPIO configuration
7. Implement preemption threshold scheduling in ThreadX

## Code Examples

### FreeRTOS Task Creation
```c
void vTask1(void *pvParameters) {
    for (;;) {
        printf("Task 1 running\n");
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

int main(void) {
    xTaskCreate(vTask1, "Task1", 128, NULL, 1, NULL);
    vTaskStartScheduler();
    for (;;); // Should never reach here
}
```

### Zephyr Thread Creation
```c
#define STACK_SIZE 1024
#define PRIORITY 5

K_THREAD_STACK_DEFINE(thread_stack, STACK_SIZE);
struct k_thread thread_data;

void thread_entry(void *p1, void *p2, void *p3) {
    while (1) {
        printk("Thread running\n");
        k_sleep(K_MSEC(1000));
    }
}

int main(void) {
    k_thread_create(&thread_data, thread_stack,
                    STACK_SIZE, thread_entry,
                    NULL, NULL, NULL,
                    PRIORITY, 0, K_NO_WAIT);
    return 0;
}
```

### ThreadX Thread Creation
```c
TX_THREAD my_thread;
UCHAR thread_stack[1024];

void thread_entry(ULONG thread_input) {
    while (1) {
        printf("Thread running\n");
        tx_thread_sleep(100); // 100 ticks
    }
}

int main(void) {
    tx_kernel_enter();
}

void tx_application_define(void *first_unused_memory) {
    tx_thread_create(&my_thread, "My Thread",
                     thread_entry, 0,
                     thread_stack, sizeof(thread_stack),
                     1, 1, TX_NO_TIME_SLICE, TX_AUTO_START);
}
```

## Resources

### FreeRTOS
- [FreeRTOS.org](https://www.freertos.org/)
- [AWS FreeRTOS](https://aws.amazon.com/freertos/)
- "Mastering the FreeRTOS Real Time Kernel" by Richard Barry

### Zephyr
- [Zephyr Project](https://www.zephyrproject.org/)
- [Zephyr Documentation](https://docs.zephyrproject.org/)
- Zephyr Developer Summit talks

### ThreadX
- [Azure RTOS ThreadX](https://azure.microsoft.com/en-us/services/rtos/)
- [ThreadX Documentation](https://docs.microsoft.com/en-us/azure/rtos/threadx/)
- "The ThreadX RTOS Advantage" whitepaper

## Community & Support

- Forums, mailing lists, Discord servers
- Stack Overflow tags
- GitHub repositories
- Commercial support options
