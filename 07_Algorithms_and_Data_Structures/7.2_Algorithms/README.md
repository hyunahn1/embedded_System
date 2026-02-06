# 7.2 Algorithms

## Topics Covered

### DSP Basics (FIR/IIR Filters, FFT)

#### Digital Signal Processing Overview
- **Purpose**: Process analog signals in digital domain
- **Applications**: Audio, sensor filtering, communication
- **Sampling**: Nyquist theorem (fs > 2 * fmax)

#### FIR Filter (Finite Impulse Response)

##### Characteristics
- **Linear Phase**: No phase distortion
- **Stable**: Always stable (no feedback)
- **Order**: Requires more taps for sharp cutoff

##### Implementation
```c
typedef struct {
    float *coefficients;  // Filter taps (b0, b1, ..., bN)
    float *state;         // Delay line (x[n-1], x[n-2], ...)
    size_t num_taps;      // Number of coefficients
    size_t index;         // Current position in delay line
} FIR_Filter_t;

void fir_init(FIR_Filter_t *fir, float *coeffs, float *state, size_t num_taps) {
    fir->coefficients = coeffs;
    fir->state = state;
    fir->num_taps = num_taps;
    fir->index = 0;
    
    // Clear state
    for (size_t i = 0; i < num_taps; i++) {
        state[i] = 0.0f;
    }
}

float fir_process(FIR_Filter_t *fir, float input) {
    // Add new sample to delay line
    fir->state[fir->index] = input;
    
    // Compute output: y[n] = b0*x[n] + b1*x[n-1] + ... + bN*x[n-N]
    float output = 0.0f;
    size_t idx = fir->index;
    
    for (size_t i = 0; i < fir->num_taps; i++) {
        output += fir->coefficients[i] * fir->state[idx];
        
        // Wrap around delay line
        if (idx == 0) {
            idx = fir->num_taps - 1;
        } else {
            idx--;
        }
    }
    
    // Update index
    fir->index = (fir->index + 1) % fir->num_taps;
    
    return output;
}
```

##### Example: Low-Pass Filter
```c
// 5-tap low-pass FIR filter (cutoff ~0.2 * fs)
float coeffs[5] = {0.1f, 0.2f, 0.4f, 0.2f, 0.1f};
float state[5];

FIR_Filter_t lpf;
fir_init(&lpf, coeffs, state, 5);

// Process samples
float filtered = fir_process(&lpf, noisy_sample);
```

#### IIR Filter (Infinite Impulse Response)

##### Characteristics
- **Feedback**: Uses previous outputs
- **Efficient**: Fewer coefficients for sharp cutoff
- **Stability**: Can be unstable if not designed carefully
- **Phase**: Non-linear phase response

##### Implementation (Direct Form I)
```c
typedef struct {
    float *a_coeffs;  // Feedback coefficients (a1, a2, ...)
    float *b_coeffs;  // Feedforward coefficients (b0, b1, ...)
    float *x_state;   // Input delay line
    float *y_state;   // Output delay line
    size_t order;     // Filter order
    size_t index;
} IIR_Filter_t;

void iir_init(IIR_Filter_t *iir, float *a, float *b, 
              float *x_state, float *y_state, size_t order) {
    iir->a_coeffs = a;
    iir->b_coeffs = b;
    iir->x_state = x_state;
    iir->y_state = y_state;
    iir->order = order;
    iir->index = 0;
    
    for (size_t i = 0; i < order; i++) {
        x_state[i] = 0.0f;
        y_state[i] = 0.0f;
    }
}

float iir_process(IIR_Filter_t *iir, float input) {
    // y[n] = b0*x[n] + b1*x[n-1] + ... - a1*y[n-1] - a2*y[n-2] - ...
    
    iir->x_state[iir->index] = input;
    
    float output = 0.0f;
    size_t idx = iir->index;
    
    // Feedforward part
    for (size_t i = 0; i < iir->order; i++) {
        output += iir->b_coeffs[i] * iir->x_state[idx];
        idx = (idx == 0) ? (iir->order - 1) : (idx - 1);
    }
    
    idx = iir->index;
    // Feedback part
    for (size_t i = 1; i < iir->order; i++) {
        idx = (idx == 0) ? (iir->order - 1) : (idx - 1);
        output -= iir->a_coeffs[i] * iir->y_state[idx];
    }
    
    iir->y_state[iir->index] = output;
    iir->index = (iir->index + 1) % iir->order;
    
    return output;
}
```

##### Biquad Filter (2nd Order IIR)
```c
// More efficient and numerically stable
typedef struct {
    float b0, b1, b2;  // Feedforward
    float a1, a2;      // Feedback
    float x1, x2;      // Input history
    float y1, y2;      // Output history
} Biquad_t;

void biquad_init(Biquad_t *bq, float b0, float b1, float b2, 
                 float a1, float a2) {
    bq->b0 = b0; bq->b1 = b1; bq->b2 = b2;
    bq->a1 = a1; bq->a2 = a2;
    bq->x1 = 0.0f; bq->x2 = 0.0f;
    bq->y1 = 0.0f; bq->y2 = 0.0f;
}

float biquad_process(Biquad_t *bq, float input) {
    float output = bq->b0 * input + bq->b1 * bq->x1 + bq->b2 * bq->x2
                   - bq->a1 * bq->y1 - bq->a2 * bq->y2;
    
    // Update history
    bq->x2 = bq->x1;
    bq->x1 = input;
    bq->y2 = bq->y1;
    bq->y1 = output;
    
    return output;
}
```

#### FFT (Fast Fourier Transform)

##### Purpose
- **Frequency Analysis**: Convert time-domain to frequency-domain
- **Applications**: Spectrum analysis, filtering, compression

##### Radix-2 FFT (Simplified)
```c
#include <math.h>
#include <complex.h>

void fft_radix2(float complex *x, size_t N) {
    // N must be power of 2
    
    // Bit-reversal permutation
    size_t j = 0;
    for (size_t i = 0; i < N - 1; i++) {
        if (i < j) {
            float complex temp = x[i];
            x[i] = x[j];
            x[j] = temp;
        }
        
        size_t k = N / 2;
        while (k <= j) {
            j -= k;
            k /= 2;
        }
        j += k;
    }
    
    // FFT computation
    for (size_t s = 1; s <= log2(N); s++) {
        size_t m = 1 << s;  // 2^s
        float complex w_m = cexp(-2.0f * M_PI * I / m);
        
        for (size_t k = 0; k < N; k += m) {
            float complex w = 1.0f;
            
            for (size_t j = 0; j < m / 2; j++) {
                float complex t = w * x[k + j + m / 2];
                float complex u = x[k + j];
                
                x[k + j] = u + t;
                x[k + j + m / 2] = u - t;
                
                w *= w_m;
            }
        }
    }
}

// Usage with ARM CMSIS-DSP (more efficient)
#include "arm_math.h"

arm_rfft_fast_instance_f32 fft_instance;
float32_t input[1024];
float32_t output[1024];

arm_rfft_fast_init_f32(&fft_instance, 1024);
arm_rfft_fast_f32(&fft_instance, input, output, 0);  // Forward FFT

// output[0] = DC component
// output[1..N/2] = real parts
// output[N/2+1..N-1] = imaginary parts
```

### CRC Calculation (Software vs Hardware)

#### CRC Overview
- **Purpose**: Error detection in data transmission
- **Polynomial**: Defines CRC algorithm (e.g., CRC-16, CRC-32)
- **Properties**: Detects burst errors, single-bit errors

#### CRC-16 (Software)
```c
// CRC-16-CCITT: Polynomial 0x1021
uint16_t crc16_ccitt(const uint8_t *data, size_t length) {
    uint16_t crc = 0xFFFF;  // Initial value
    
    for (size_t i = 0; i < length; i++) {
        crc ^= (uint16_t)data[i] << 8;
        
        for (int j = 0; j < 8; j++) {
            if (crc & 0x8000) {
                crc = (crc << 1) ^ 0x1021;
            } else {
                crc <<= 1;
            }
        }
    }
    
    return crc;
}
```

#### CRC-32 with Lookup Table
```c
// Precomputed lookup table for CRC-32 (Ethernet polynomial)
static uint32_t crc32_table[256];

void crc32_init_table(void) {
    for (uint32_t i = 0; i < 256; i++) {
        uint32_t crc = i;
        for (int j = 0; j < 8; j++) {
            if (crc & 1) {
                crc = (crc >> 1) ^ 0xEDB88320;
            } else {
                crc >>= 1;
            }
        }
        crc32_table[i] = crc;
    }
}

uint32_t crc32(const uint8_t *data, size_t length) {
    uint32_t crc = 0xFFFFFFFF;
    
    for (size_t i = 0; i < length; i++) {
        uint8_t index = (crc ^ data[i]) & 0xFF;
        crc = (crc >> 8) ^ crc32_table[index];
    }
    
    return ~crc;
}
```

#### Hardware CRC (STM32 Example)
```c
#include "stm32f4xx.h"

uint32_t crc32_hardware(const uint32_t *data, size_t length) {
    // Enable CRC peripheral clock
    RCC->AHB1ENR |= RCC_AHB1ENR_CRCEN;
    
    // Reset CRC
    CRC->CR = CRC_CR_RESET;
    
    // Feed data
    for (size_t i = 0; i < length; i++) {
        CRC->DR = data[i];
    }
    
    return CRC->DR;
}
```

### PID Control (General Implementation)

#### PID Controller Basics
- **Proportional**: Current error (Kp * error)
- **Integral**: Accumulated error (Ki * ∫error dt)
- **Derivative**: Rate of change (Kd * d(error)/dt)

#### PID Implementation
```c
typedef struct {
    float Kp;           // Proportional gain
    float Ki;           // Integral gain
    float Kd;           // Derivative gain
    float setpoint;     // Desired value
    float integral;     // Accumulated integral
    float prev_error;   // Previous error for derivative
    float dt;           // Time step
    float output_min;   // Output limits
    float output_max;
} PID_Controller_t;

void pid_init(PID_Controller_t *pid, float Kp, float Ki, float Kd, 
              float dt, float out_min, float out_max) {
    pid->Kp = Kp;
    pid->Ki = Ki;
    pid->Kd = Kd;
    pid->dt = dt;
    pid->output_min = out_min;
    pid->output_max = out_max;
    pid->integral = 0.0f;
    pid->prev_error = 0.0f;
}

void pid_set_setpoint(PID_Controller_t *pid, float setpoint) {
    pid->setpoint = setpoint;
}

float pid_compute(PID_Controller_t *pid, float measurement) {
    // Calculate error
    float error = pid->setpoint - measurement;
    
    // Proportional term
    float p_term = pid->Kp * error;
    
    // Integral term
    pid->integral += error * pid->dt;
    float i_term = pid->Ki * pid->integral;
    
    // Derivative term
    float derivative = (error - pid->prev_error) / pid->dt;
    float d_term = pid->Kd * derivative;
    
    // Compute output
    float output = p_term + i_term + d_term;
    
    // Clamp output to limits
    if (output > pid->output_max) {
        output = pid->output_max;
    } else if (output < pid->output_min) {
        output = pid->output_min;
    }
    
    // Save error for next iteration
    pid->prev_error = error;
    
    return output;
}

// Reset integral (to prevent windup)
void pid_reset(PID_Controller_t *pid) {
    pid->integral = 0.0f;
    pid->prev_error = 0.0f;
}
```

#### PID with Anti-Windup
```c
float pid_compute_antiwindup(PID_Controller_t *pid, float measurement) {
    float error = pid->setpoint - measurement;
    
    // Proportional
    float p_term = pid->Kp * error;
    
    // Derivative
    float derivative = (error - pid->prev_error) / pid->dt;
    float d_term = pid->Kd * derivative;
    
    // Tentative output (without integral)
    float output_no_i = p_term + d_term;
    
    // Integral with anti-windup
    // Only integrate if output is not saturated
    if ((output_no_i < pid->output_max && output_no_i > pid->output_min) ||
        (output_no_i >= pid->output_max && error < 0) ||
        (output_no_i <= pid->output_min && error > 0)) {
        pid->integral += error * pid->dt;
    }
    
    float i_term = pid->Ki * pid->integral;
    float output = output_no_i + i_term;
    
    // Clamp
    if (output > pid->output_max) {
        output = pid->output_max;
    } else if (output < pid->output_min) {
        output = pid->output_min;
    }
    
    pid->prev_error = error;
    return output;
}
```

#### Example: Temperature Control
```c
// Control heater to maintain 50°C
PID_Controller_t temp_pid;
pid_init(&temp_pid, 2.0f, 0.5f, 1.0f, 0.1f, 0.0f, 100.0f);
pid_set_setpoint(&temp_pid, 50.0f);

// In control loop (every 100ms)
float temperature = read_temperature_sensor();
float heater_power = pid_compute(&temp_pid, temperature);
set_heater_pwm((uint8_t)heater_power);
```

## Key Concepts

- **FIR**: Linear phase, always stable
- **IIR**: Efficient, can be unstable
- **FFT**: O(N log N) vs O(N²) for DFT
- **CRC**: Polynomial-based error detection
- **PID**: Proportional + Integral + Derivative control

## Practical Exercises

1. Implement FIR low-pass filter for sensor
2. Design IIR filter with biquad sections
3. Perform FFT on audio samples
4. Calculate CRC for packet validation
5. Tune PID controller for motor speed
6. Compare software vs hardware CRC performance
7. Implement anti-windup PID

## Performance Optimization

- **ARM CMSIS-DSP**: Optimized DSP library
- **Fixed-Point**: Use integer math instead of float
- **SIMD**: Use ARM NEON for parallel processing
- **Hardware CRC**: Much faster than software

## Best Practices

1. **Test filters** with known signals
2. **Check stability** of IIR filters
3. **Tune PID** systematically (Ziegler-Nichols)
4. **Use lookup tables** for CRC
5. **Profile performance** before optimizing
6. **Validate with reference** implementations

## Resources

- ARM CMSIS-DSP library
- "Digital Signal Processing" by Proakis & Manolakis
- "Embedded Systems: Real-Time Interfacing" by Valvano
- PID tuning guides
- CRC specifications
