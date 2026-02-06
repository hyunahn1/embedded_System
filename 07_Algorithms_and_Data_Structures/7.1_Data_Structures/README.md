# 7.1 Data Structures

## Topics Covered

### Circular Buffers (FIFO)

#### Overview
- **Purpose**: Fixed-size FIFO queue
- **Use Cases**: 
  - UART buffers
  - Audio streaming
  - Sensor data buffering
  - Inter-task communication

#### Basic Implementation
```c
typedef struct {
    uint8_t *buffer;      // Data buffer
    size_t head;          // Write index
    size_t tail;          // Read index
    size_t size;          // Buffer size
    size_t count;         // Number of elements
} CircularBuffer_t;

void cbuf_init(CircularBuffer_t *cb, uint8_t *buffer, size_t size) {
    cb->buffer = buffer;
    cb->head = 0;
    cb->tail = 0;
    cb->size = size;
    cb->count = 0;
}

bool cbuf_put(CircularBuffer_t *cb, uint8_t data) {
    if (cb->count >= cb->size) {
        return false;  // Buffer full
    }
    
    cb->buffer[cb->head] = data;
    cb->head = (cb->head + 1) % cb->size;
    cb->count++;
    
    return true;
}

bool cbuf_get(CircularBuffer_t *cb, uint8_t *data) {
    if (cb->count == 0) {
        return false;  // Buffer empty
    }
    
    *data = cb->buffer[cb->tail];
    cb->tail = (cb->tail + 1) % cb->size;
    cb->count--;
    
    return true;
}

size_t cbuf_available(CircularBuffer_t *cb) {
    return cb->count;
}

size_t cbuf_free_space(CircularBuffer_t *cb) {
    return cb->size - cb->count;
}
```

#### Power-of-2 Optimization
```c
// Use bitwise AND instead of modulo (faster)
typedef struct {
    uint8_t *buffer;
    size_t head;
    size_t tail;
    size_t mask;  // size - 1 (for power of 2 sizes)
} FastCircularBuffer_t;

void fcbuf_init(FastCircularBuffer_t *cb, uint8_t *buffer, size_t size) {
    // size must be power of 2
    assert((size & (size - 1)) == 0);
    
    cb->buffer = buffer;
    cb->head = 0;
    cb->tail = 0;
    cb->mask = size - 1;
}

bool fcbuf_put(FastCircularBuffer_t *cb, uint8_t data) {
    size_t next_head = (cb->head + 1) & cb->mask;
    
    if (next_head == cb->tail) {
        return false;  // Full
    }
    
    cb->buffer[cb->head] = data;
    cb->head = next_head;
    
    return true;
}

bool fcbuf_get(FastCircularBuffer_t *cb, uint8_t *data) {
    if (cb->head == cb->tail) {
        return false;  // Empty
    }
    
    *data = cb->buffer[cb->tail];
    cb->tail = (cb->tail + 1) & cb->mask;
    
    return true;
}
```

#### Lock-Free Circular Buffer (Single Producer, Single Consumer)
```c
typedef struct {
    volatile uint8_t *buffer;
    volatile size_t head;  // Written by producer
    volatile size_t tail;  // Written by consumer
    size_t mask;
} LockFreeBuffer_t;

// Producer (ISR or task)
bool lfbuf_put(LockFreeBuffer_t *cb, uint8_t data) {
    size_t head = cb->head;
    size_t next_head = (head + 1) & cb->mask;
    
    if (next_head == cb->tail) {
        return false;  // Full
    }
    
    cb->buffer[head] = data;
    __DMB();  // Data memory barrier
    cb->head = next_head;
    
    return true;
}

// Consumer (ISR or task)
bool lfbuf_get(LockFreeBuffer_t *cb, uint8_t *data) {
    size_t tail = cb->tail;
    
    if (cb->head == tail) {
        return false;  // Empty
    }
    
    *data = cb->buffer[tail];
    __DMB();  // Data memory barrier
    cb->tail = (tail + 1) & cb->mask;
    
    return true;
}
```

### Linked Lists (Intrusive Lists in Kernel)

#### Standard Linked List
```c
typedef struct Node {
    void *data;
    struct Node *next;
} Node_t;

typedef struct {
    Node_t *head;
    size_t count;
} LinkedList_t;

void list_init(LinkedList_t *list) {
    list->head = NULL;
    list->count = 0;
}

void list_add(LinkedList_t *list, void *data) {
    Node_t *node = malloc(sizeof(Node_t));
    node->data = data;
    node->next = list->head;
    list->head = node;
    list->count++;
}

void *list_remove_first(LinkedList_t *list) {
    if (list->head == NULL) {
        return NULL;
    }
    
    Node_t *node = list->head;
    void *data = node->data;
    list->head = node->next;
    free(node);
    list->count--;
    
    return data;
}
```

#### Intrusive Linked List (Linux Kernel Style)
```c
// List node embedded in data structure
typedef struct ListNode {
    struct ListNode *next;
    struct ListNode *prev;
} ListNode_t;

#define LIST_HEAD_INIT(name) { &(name), &(name) }

static inline void list_init(ListNode_t *head) {
    head->next = head;
    head->prev = head;
}

static inline void list_add(ListNode_t *new_node, ListNode_t *head) {
    new_node->next = head->next;
    new_node->prev = head;
    head->next->prev = new_node;
    head->next = new_node;
}

static inline void list_remove(ListNode_t *node) {
    node->prev->next = node->next;
    node->next->prev = node->prev;
    node->next = node;
    node->prev = node;
}

// Container_of macro to get parent structure
#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))

// Example usage
typedef struct {
    int id;
    char name[32];
    ListNode_t list;  // Embedded list node
} Device_t;

ListNode_t device_list = LIST_HEAD_INIT(device_list);

void add_device(Device_t *dev) {
    list_add(&dev->list, &device_list);
}

void iterate_devices(void) {
    ListNode_t *node;
    for (node = device_list.next; node != &device_list; node = node->next) {
        Device_t *dev = container_of(node, Device_t, list);
        printf("Device: %s (ID: %d)\n", dev->name, dev->id);
    }
}
```

### Bit Maps & Bit Fields

#### Bit Manipulation Macros
```c
// Set/clear/toggle/test bits
#define BIT(n)              (1U << (n))
#define SET_BIT(reg, bit)   ((reg) |= BIT(bit))
#define CLEAR_BIT(reg, bit) ((reg) &= ~BIT(bit))
#define TOGGLE_BIT(reg, bit) ((reg) ^= BIT(bit))
#define TEST_BIT(reg, bit)  (((reg) & BIT(bit)) != 0)

// Multi-bit operations
#define BITS(start, end)    ((BIT((end) - (start) + 1) - 1) << (start))
#define SET_BITS(reg, mask, val) \
    ((reg) = ((reg) & ~(mask)) | ((val) & (mask)))
#define GET_BITS(reg, mask)  ((reg) & (mask))
```

#### Bitmap for Resource Allocation
```c
typedef struct {
    uint32_t *bitmap;
    size_t num_bits;
} Bitmap_t;

void bitmap_init(Bitmap_t *bm, uint32_t *buffer, size_t num_bits) {
    bm->bitmap = buffer;
    bm->num_bits = num_bits;
    memset(buffer, 0, (num_bits + 31) / 32 * sizeof(uint32_t));
}

void bitmap_set(Bitmap_t *bm, size_t bit) {
    if (bit < bm->num_bits) {
        bm->bitmap[bit / 32] |= (1U << (bit % 32));
    }
}

void bitmap_clear(Bitmap_t *bm, size_t bit) {
    if (bit < bm->num_bits) {
        bm->bitmap[bit / 32] &= ~(1U << (bit % 32));
    }
}

bool bitmap_test(Bitmap_t *bm, size_t bit) {
    if (bit < bm->num_bits) {
        return (bm->bitmap[bit / 32] & (1U << (bit % 32))) != 0;
    }
    return false;
}

// Find first set bit (allocate resource)
int bitmap_find_first_set(Bitmap_t *bm) {
    for (size_t i = 0; i < (bm->num_bits + 31) / 32; i++) {
        if (bm->bitmap[i] != 0) {
            // Find bit position using count leading zeros
            int bit = __builtin_ctz(bm->bitmap[i]);
            return i * 32 + bit;
        }
    }
    return -1;  // No set bit found
}

// Find first clear bit (allocate resource)
int bitmap_find_first_clear(Bitmap_t *bm) {
    for (size_t i = 0; i < (bm->num_bits + 31) / 32; i++) {
        if (bm->bitmap[i] != 0xFFFFFFFF) {
            int bit = __builtin_ctz(~bm->bitmap[i]);
            size_t result = i * 32 + bit;
            if (result < bm->num_bits) {
                return result;
            }
        }
    }
    return -1;
}
```

#### Bit Fields in Structures
```c
// Compact representation
typedef struct {
    uint32_t enabled     : 1;   // 1 bit
    uint32_t mode        : 3;   // 3 bits (0-7)
    uint32_t priority    : 4;   // 4 bits (0-15)
    uint32_t reserved    : 24;  // 24 bits
} DeviceConfig_t;

// Usage
DeviceConfig_t config = {0};
config.enabled = 1;
config.mode = 5;
config.priority = 10;

// Note: Bit fields are compiler-dependent
// For portability, use bit manipulation instead:

typedef struct {
    uint32_t flags;
} PortableConfig_t;

#define CONFIG_ENABLED_MASK     0x00000001
#define CONFIG_MODE_MASK        0x0000000E
#define CONFIG_MODE_SHIFT       1
#define CONFIG_PRIORITY_MASK    0x000000F0
#define CONFIG_PRIORITY_SHIFT   4

void set_mode(PortableConfig_t *cfg, uint8_t mode) {
    cfg->flags = (cfg->flags & ~CONFIG_MODE_MASK) | 
                 ((mode << CONFIG_MODE_SHIFT) & CONFIG_MODE_MASK);
}

uint8_t get_mode(PortableConfig_t *cfg) {
    return (cfg->flags & CONFIG_MODE_MASK) >> CONFIG_MODE_SHIFT;
}
```

### Additional Data Structures

#### Fixed-Size Stack
```c
typedef struct {
    int *data;
    size_t top;
    size_t capacity;
} Stack_t;

void stack_init(Stack_t *stack, int *buffer, size_t capacity) {
    stack->data = buffer;
    stack->top = 0;
    stack->capacity = capacity;
}

bool stack_push(Stack_t *stack, int value) {
    if (stack->top >= stack->capacity) {
        return false;  // Stack full
    }
    stack->data[stack->top++] = value;
    return true;
}

bool stack_pop(Stack_t *stack, int *value) {
    if (stack->top == 0) {
        return false;  // Stack empty
    }
    *value = stack->data[--stack->top];
    return true;
}
```

#### Priority Queue (Binary Heap)
```c
typedef struct {
    int *heap;
    size_t size;
    size_t capacity;
} PriorityQueue_t;

void pq_init(PriorityQueue_t *pq, int *buffer, size_t capacity) {
    pq->heap = buffer;
    pq->size = 0;
    pq->capacity = capacity;
}

static void swap(int *a, int *b) {
    int temp = *a;
    *a = *b;
    *b = temp;
}

static void heapify_up(PriorityQueue_t *pq, size_t index) {
    while (index > 0) {
        size_t parent = (index - 1) / 2;
        if (pq->heap[index] <= pq->heap[parent]) {
            break;
        }
        swap(&pq->heap[index], &pq->heap[parent]);
        index = parent;
    }
}

static void heapify_down(PriorityQueue_t *pq, size_t index) {
    while (2 * index + 1 < pq->size) {
        size_t left = 2 * index + 1;
        size_t right = 2 * index + 2;
        size_t largest = index;
        
        if (pq->heap[left] > pq->heap[largest]) {
            largest = left;
        }
        if (right < pq->size && pq->heap[right] > pq->heap[largest]) {
            largest = right;
        }
        
        if (largest == index) {
            break;
        }
        
        swap(&pq->heap[index], &pq->heap[largest]);
        index = largest;
    }
}

bool pq_insert(PriorityQueue_t *pq, int value) {
    if (pq->size >= pq->capacity) {
        return false;
    }
    
    pq->heap[pq->size] = value;
    heapify_up(pq, pq->size);
    pq->size++;
    
    return true;
}

bool pq_extract_max(PriorityQueue_t *pq, int *value) {
    if (pq->size == 0) {
        return false;
    }
    
    *value = pq->heap[0];
    pq->heap[0] = pq->heap[--pq->size];
    heapify_down(pq, 0);
    
    return true;
}
```

## Key Concepts

- **Circular Buffer**: Essential for streaming data
- **Intrusive List**: Zero-allocation linked list
- **Bitmap**: Space-efficient set representation
- **Fixed-Size**: Predictable memory usage

## Practical Exercises

1. Implement circular buffer for UART
2. Create intrusive list for task management
3. Use bitmap for resource allocation
4. Build priority queue for event scheduling
5. Optimize circular buffer with power-of-2 size
6. Implement lock-free ring buffer
7. Create bit field parser for protocol

## Performance Considerations

- **Circular Buffer**: O(1) insert/remove
- **Linked List**: O(1) insert/remove (with pointer)
- **Bitmap**: O(1) set/clear, O(n) find
- **Binary Heap**: O(log n) insert/extract

## Best Practices

1. **Use static allocation** when possible
2. **Power-of-2 sizes** for circular buffers
3. **Cache-friendly** data layout
4. **Avoid dynamic allocation** in ISRs
5. **Test boundary conditions**
6. **Use intrusive lists** to avoid allocation

## Resources

- Linux kernel data structures
- "Embedded Systems" by Raj Kamal
- ARM CMSIS library
- Data structures textbooks
