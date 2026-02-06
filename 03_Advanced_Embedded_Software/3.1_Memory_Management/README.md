# 3.1 Memory Management & Optimization

## Topics Covered

### Static vs Dynamic Allocation

#### Static Allocation
- **Characteristics**
  - Memory allocated at compile-time
  - Fixed size and location
  - No fragmentation
  - Predictable behavior
  - Zero runtime overhead

- **Use Cases**
  - Safety-critical systems
  - Real-time systems with hard deadlines
  - Resource-constrained environments
  - When maximum memory usage is known

- **Techniques**
  - Global and static variables
  - Compile-time arrays
  - Linker-allocated sections

#### Dynamic Allocation
- **Characteristics**
  - Memory allocated at runtime
  - Flexible sizing
  - Risk of fragmentation
  - Can fail (out of memory)
  - Runtime overhead

- **Challenges in Embedded**
  - Heap fragmentation
  - Unpredictable allocation time
  - Memory leaks
  - Limited heap size

- **Safe Dynamic Allocation**
  - Memory pools (fixed-size blocks)
  - Block allocators
  - Custom allocators with guarantees

#### Memory Pools
- **Fixed-Size Blocks**
  - Pre-allocated blocks of uniform size
  - Fast allocation/deallocation (O(1))
  - No fragmentation
  - Some waste if sizes vary

- **Implementation**
  ```c
  typedef struct MemBlock {
      struct MemBlock *next;
  } MemBlock_t;
  
  typedef struct {
      MemBlock_t *free_list;
      uint8_t *pool;
      size_t block_size;
      size_t num_blocks;
  } MemPool_t;
  ```

- **Use Cases**
  - Message buffers
  - Fixed-size packets
  - Object pools

#### Variable Block Allocators
- **Buddy System**
  - Powers-of-two block sizes
  - Fast coalescing
  - Some internal fragmentation

- **Segregated Free Lists**
  - Multiple free lists by size class
  - Fast allocation for common sizes

### Buffer Management

#### Ring Buffers (Circular Buffers)
- **Characteristics**
  - FIFO behavior
  - Fixed size
  - Wrap-around indexing
  - No memory allocation/deallocation

- **Implementation**
  ```c
  typedef struct {
      uint8_t *buffer;
      size_t head;
      size_t tail;
      size_t size;
      size_t count;
  } RingBuffer_t;
  ```

- **Use Cases**
  - UART receive/transmit buffers
  - Audio/video streaming
  - Sensor data buffering
  - Communication protocols

- **Lock-Free Variants**
  - Single producer, single consumer
  - Atomic operations for thread-safety
  - Cache-line padding to avoid false sharing

#### Zero-Copy Techniques
- **Concept**
  - Avoid copying data in memory
  - Pass pointers/references instead
  - Share buffers between layers

- **Methods**
  - Scatter-gather DMA
  - Buffer ownership transfer
  - Memory-mapped buffers
  - Reference counting

- **Challenges**
  - Buffer lifetime management
  - Synchronization
  - Cache coherency

- **Benefits**
  - Reduced CPU usage
  - Lower latency
  - Better throughput

### Cache Optimization

#### Cache Basics
- **Types**
  - Instruction cache (I-cache)
  - Data cache (D-cache)
  - Unified cache

- **Parameters**
  - Cache line size (32, 64 bytes typical)
  - Associativity (direct-mapped, N-way set-associative)
  - Write policy (write-through, write-back)

#### Data Cache Optimization
- **Data Alignment**
  - Align structures to cache line boundaries
  - Prevent false sharing in multi-core
  - Pad structures to avoid split accesses

- **Access Patterns**
  - Sequential access (cache-friendly)
  - Stride patterns
  - Random access (cache-unfriendly)

- **Data Layout**
  - Array of Structures (AoS) vs Structure of Arrays (SoA)
  - Hot/cold data separation
  - Prefetching hints

#### Instruction Cache Optimization
- **Code Placement**
  - Group frequently-called functions
  - Inline small, hot functions
  - Avoid code size bloat

- **Branch Prediction**
  - Predictable branches
  - Minimize branches in hot paths
  - Profile-guided optimization

#### Cache Coherency
- **Multi-core Issues**
  - MESI protocol (Modified, Exclusive, Shared, Invalid)
  - Cache line ping-pong
  - False sharing

- **DMA and Cache**
  - Cache invalidation before DMA read
  - Cache flushing before DMA write
  - Non-cacheable memory regions
  - Memory barriers

- **Cache Maintenance Operations**
  ```c
  // Invalidate D-cache (before reading DMA data)
  SCB_InvalidateDCache_by_Addr(buffer, size);
  
  // Clean D-cache (before DMA writes data out)
  SCB_CleanDCache_by_Addr(buffer, size);
  ```

### Code Size vs Speed Trade-offs

#### Code Size Optimization
- **Compiler Flags**
  - `-Os`: Optimize for size
  - `-ffunction-sections`, `-fdata-sections`: Section per function
  - `--gc-sections`: Remove unused sections

- **Techniques**
  - Function outlining
  - Avoid inline for large functions
  - Use loops instead of unrolling
  - Share common code sequences

- **Tools**
  - Size profiling (size, nm, readelf)
  - Bloaty McBloatface
  - Map file analysis

#### Speed Optimization
- **Compiler Flags**
  - `-O2`, `-O3`: Optimize for speed
  - `-ffast-math`: Relaxed IEEE compliance
  - `-funroll-loops`: Loop unrolling

- **Techniques**
  - Inline hot functions
  - Loop unrolling
  - Avoid function calls in tight loops
  - Use lookup tables vs computation

- **Profiling**
  - Identify hot spots (gprof, perf)
  - Focus optimization efforts
  - Measure, don't guess

#### Balanced Approach
- **Hot/Cold Code Separation**
  - Optimize hot paths for speed
  - Optimize cold paths for size
  - Profile to identify hot/cold regions

- **Strategic Inlining**
  - Inline small, frequently-called functions
  - Don't inline large or rarely-called functions

## Key Concepts

- **Memory Fragmentation**: Internal vs External
- **Memory Alignment**: Natural alignment, packed structures
- **Cache Locality**: Temporal vs Spatial
- **Working Set**: Active memory footprint

## Practical Exercises

1. Implement a memory pool allocator
2. Create a lock-free ring buffer
3. Analyze cache miss rates with different data layouts
4. Optimize a function for code size
5. Profile and optimize a hot path
6. Implement zero-copy buffer management
7. Debug memory corruption with MPU

## Code Examples

### Memory Pool Implementation
```c
void mempool_init(MemPool_t *pool, void *memory, 
                  size_t block_size, size_t num_blocks) {
    pool->pool = (uint8_t *)memory;
    pool->block_size = block_size;
    pool->num_blocks = num_blocks;
    pool->free_list = NULL;
    
    // Initialize free list
    for (size_t i = 0; i < num_blocks; i++) {
        MemBlock_t *block = (MemBlock_t *)(pool->pool + i * block_size);
        block->next = pool->free_list;
        pool->free_list = block;
    }
}

void *mempool_alloc(MemPool_t *pool) {
    if (pool->free_list == NULL) {
        return NULL; // Out of memory
    }
    
    MemBlock_t *block = pool->free_list;
    pool->free_list = block->next;
    return (void *)block;
}

void mempool_free(MemPool_t *pool, void *ptr) {
    MemBlock_t *block = (MemBlock_t *)ptr;
    block->next = pool->free_list;
    pool->free_list = block;
}
```

### Ring Buffer Implementation
```c
bool ringbuf_put(RingBuffer_t *rb, uint8_t data) {
    if (rb->count >= rb->size) {
        return false; // Buffer full
    }
    
    rb->buffer[rb->head] = data;
    rb->head = (rb->head + 1) % rb->size;
    rb->count++;
    return true;
}

bool ringbuf_get(RingBuffer_t *rb, uint8_t *data) {
    if (rb->count == 0) {
        return false; // Buffer empty
    }
    
    *data = rb->buffer[rb->tail];
    rb->tail = (rb->tail + 1) % rb->size;
    rb->count--;
    return true;
}
```

### Cache-Aligned Structure
```c
#define CACHE_LINE_SIZE 32

// Align structure to cache line
typedef struct __attribute__((aligned(CACHE_LINE_SIZE))) {
    volatile uint32_t producer_index;
    uint8_t padding1[CACHE_LINE_SIZE - sizeof(uint32_t)];
    
    volatile uint32_t consumer_index;
    uint8_t padding2[CACHE_LINE_SIZE - sizeof(uint32_t)];
    
    uint8_t data[256];
} LockFreeQueue_t;
```

## Performance Metrics

- **Allocation time**: Static: 0, Pool: O(1), Malloc: O(n)
- **Fragmentation**: Static: 0%, Pool: <20%, Malloc: varies
- **Cache hit rate**: >95% is good for data, >98% for instructions
- **Code size**: Typical embedded: 16KB-512KB

## Best Practices

1. **Prefer static allocation in embedded systems**
2. **Use memory pools for dynamic allocation**
3. **Align data structures to cache lines**
4. **Measure before optimizing**
5. **Optimize hot paths, not cold paths**
6. **Use const to place data in flash**
7. **Profile memory usage during development**

## Tools

- **Valgrind**: Memory debugging (desktop simulation)
- **AddressSanitizer**: Memory error detection
- **Heaptrack**: Heap profiler
- **Cachegrind**: Cache profiler
- **Perf**: Linux performance analyzer
- **Ozone/SystemView**: Embedded profilers

## References

- "Embedded Systems Architecture" by Daniele Lacamera
- "Making Embedded Systems" by Elecia White
- ARM Cortex-M cache documentation
- Compiler optimization guides
