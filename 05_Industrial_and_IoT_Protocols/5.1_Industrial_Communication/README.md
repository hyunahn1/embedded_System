# 5.1 Industrial Communication

## Topics Covered

### Modbus (RTU/TCP)

#### Overview
- **Origin**: Developed by Modicon (now Schneider Electric) in 1979
- **Ubiquity**: De facto standard in industrial automation
- **Variants**: Modbus RTU (serial), Modbus TCP (Ethernet), Modbus ASCII

#### Modbus RTU (Serial)

##### Physical Layer
- **Media**: RS-232, RS-485 (most common)
- **Topology**: Master-slave (1 master, up to 247 slaves)
- **Baud Rates**: 9600, 19200, 38400, 57600, 115200 bps
- **Frame Format**: 8N1, 8E1, 8O1 typical

##### Frame Structure
```
[Slave Address][Function Code][Data][CRC-16]
     1 byte         1 byte      N bytes  2 bytes
```

##### Function Codes
- **0x01**: Read Coils (digital outputs)
- **0x02**: Read Discrete Inputs (digital inputs)
- **0x03**: Read Holding Registers (16-bit read/write)
- **0x04**: Read Input Registers (16-bit read-only)
- **0x05**: Write Single Coil
- **0x06**: Write Single Register
- **0x0F**: Write Multiple Coils
- **0x10**: Write Multiple Registers

##### Example: Read Holding Registers
```c
// Request to slave 0x01: Read 2 registers starting at address 0x0000
uint8_t request[] = {
    0x01,       // Slave address
    0x03,       // Function code (Read Holding Registers)
    0x00, 0x00, // Starting address (high, low)
    0x00, 0x02, // Number of registers (high, low)
    0xC4, 0x0B  // CRC-16 (calculated)
};

// Response
uint8_t response[] = {
    0x01,       // Slave address
    0x03,       // Function code
    0x04,       // Byte count (2 registers * 2 bytes)
    0x12, 0x34, // Register 0 value
    0x56, 0x78, // Register 1 value
    0xXX, 0xXX  // CRC-16
};
```

##### CRC-16 Calculation
```c
uint16_t modbus_crc16(const uint8_t *data, size_t length) {
    uint16_t crc = 0xFFFF;
    
    for (size_t i = 0; i < length; i++) {
        crc ^= data[i];
        
        for (int j = 0; j < 8; j++) {
            if (crc & 0x0001) {
                crc = (crc >> 1) ^ 0xA001;
            } else {
                crc >>= 1;
            }
        }
    }
    
    return crc;
}
```

#### Modbus TCP

##### Frame Structure
```
[MBAP Header][Function Code][Data]
   7 bytes       1 byte      N bytes

MBAP Header:
[Transaction ID][Protocol ID][Length][Unit ID]
    2 bytes       2 bytes    2 bytes  1 byte
```

##### Example: Read Holding Registers
```c
// Modbus TCP request
uint8_t request[] = {
    0x00, 0x01, // Transaction ID
    0x00, 0x00, // Protocol ID (always 0 for Modbus)
    0x00, 0x06, // Length (bytes following)
    0x01,       // Unit ID (slave address)
    0x03,       // Function code
    0x00, 0x00, // Starting address
    0x00, 0x02  // Number of registers
};
```

##### TCP Communication
```c
#include <sys/socket.h>
#include <netinet/in.h>

int modbus_tcp_connect(const char *ip, uint16_t port) {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);  // Default: 502
    inet_pton(AF_INET, ip, &addr.sin_addr);
    
    connect(sock, (struct sockaddr *)&addr, sizeof(addr));
    
    return sock;
}

void modbus_tcp_read_registers(int sock, uint16_t addr, uint16_t count) {
    uint8_t request[12];
    // Build request frame
    
    send(sock, request, sizeof(request), 0);
    
    uint8_t response[256];
    recv(sock, response, sizeof(response), 0);
    
    // Parse response
}
```

### Profibus / Profinet

#### Profibus

##### Overview
- **Origin**: German standard (DIN 19245)
- **Variants**: Profibus DP (Decentralized Periphery), Profibus PA (Process Automation)
- **Physical Layer**: RS-485 (DP), MBP (PA)
- **Topology**: Bus, up to 127 nodes
- **Speed**: 9.6 kbps to 12 Mbps

##### Features
- **Deterministic**: Real-time communication
- **Cyclical/Acyclical**: Regular data exchange + events
- **Master-Slave**: Multiple masters allowed

##### GSD Files
- **General Station Description**: Device capability description
- **XML Format**: Hardware and communication parameters

#### Profinet

##### Overview
- **Profinet IO**: Real-time Industrial Ethernet
- **Based on Ethernet**: IEEE 802.3
- **Real-Time Classes**:
  - **RT (Real-Time)**: <10ms cycle time
  - **IRT (Isochronous Real-Time)**: <1ms, time-slotted

##### Architecture
```
┌─────────────────────────┐
│   IO Controller (PLC)   │
└───────────┬─────────────┘
            │ Ethernet
      ┌─────┴─────┬──────────┐
      │           │          │
┌─────▼────┐ ┌───▼────┐ ┌──▼─────┐
│IO Device │ │IO Device│ │IO Device│
│ (Sensor) │ │ (Motor) │ │ (Drive) │
└──────────┘ └─────────┘ └────────┘
```

##### GSDML Files
- **Device Description**: XML-based
- **Configuration**: Engineering tools import GSDML

### EtherCAT

#### Overview
- **EtherCAT**: Ethernet for Control Automation Technology
- **Developer**: Beckhoff Automation
- **Performance**: Ultra-high speed, low latency (<100μs)
- **Topology**: Line, tree, or star (no switches needed)

#### Principle
- **Process on the Fly**: Each slave device processes frame as it passes
- **Full Duplex Ethernet**: Data read on downstream, write on upstream

```
Master ───[Data]──> Slave1 ───> Slave2 ───> ... ───> SlaveN
       <──[Data]───        <───        <───     <───
```

#### Frame Structure
- **Ethernet Frame**: Standard Ethernet II or 802.3
- **EtherCAT Header**: 2 bytes
- **Datagrams**: Multiple datagrams per frame
- **Working Counter**: Verification mechanism

#### Features
- **Distributed Clocks**: Synchronization <1μs
- **Flexible Topology**: No expensive switches
- **Cost-Effective**: Standard Ethernet PHY
- **High Performance**: 1000+ I/O points in <100μs

#### Implementation
- **Master Stack**: Open-source (SOEM, IGH EtherCAT Master)
- **Slave Stack**: SSC (Slave Stack Code) from EtherCAT Technology Group

```c
// Example with SOEM (Simple Open EtherCAT Master)
#include "ethercat.h"

int main(void) {
    // Initialize EtherCAT
    if (ec_init("eth0")) {
        printf("EtherCAT initialized on eth0\n");
        
        // Find and configure slaves
        if (ec_config_init(FALSE) > 0) {
            printf("Found %d slaves\n", ec_slavecount);
            
            // Configure slaves
            ec_config_map(&IOmap);
            ec_configdc();
            
            // Set operational state
            ec_statecheck(0, EC_STATE_OPERATIONAL, EC_TIMEOUTSTATE);
            
            // Cyclic process data exchange
            while (1) {
                ec_send_processdata();
                ec_receive_processdata(EC_TIMEOUTRET);
                
                // Access process data in IOmap
            }
        }
    }
    
    ec_close();
    return 0;
}
```

### OPC UA (Industrial IoT)

#### Overview
- **OPC**: Open Platform Communications
- **OPC UA**: Unified Architecture (OPC successor)
- **Purpose**: Platform-independent industrial communication
- **Transport**: TCP/IP, Web Services (SOAP), MQTT

#### Architecture
- **Client-Server**: Traditional model
- **Publish-Subscribe**: Event-driven (MQTT)
- **Information Model**: Object-oriented, hierarchical

#### Features
- **Security**: Encryption, authentication, authorization
- **Platform-Independent**: Windows, Linux, embedded
- **Information Modeling**: Rich data structures
- **Historical Access**: Time-series data
- **Alarms & Events**: Condition monitoring

#### Information Model
```
Root
├── Objects
│   ├── Server
│   └── MyDevice
│       ├── Temperature (Variable)
│       ├── Pressure (Variable)
│       └── StartProcess (Method)
├── Types
└── Views
```

#### Example with open62541
```c
#include <open62541/server.h>

int main(void) {
    UA_Server *server = UA_Server_new();
    UA_ServerConfig_setDefault(UA_Server_getConfig(server));
    
    // Add a variable node
    UA_VariableAttributes attr = UA_VariableAttributes_default;
    UA_Int32 temperature = 25;
    UA_Variant_setScalar(&attr.value, &temperature, &UA_TYPES[UA_TYPES_INT32]);
    attr.description = UA_LOCALIZEDTEXT("en-US", "Temperature");
    attr.displayName = UA_LOCALIZEDTEXT("en-US", "Temperature");
    attr.accessLevel = UA_ACCESSLEVELMASK_READ | UA_ACCESSLEVELMASK_WRITE;
    
    UA_NodeId temperatureNodeId = 
        UA_NODEID_STRING(1, "Temperature");
    UA_QualifiedName temperatureName = 
        UA_QUALIFIEDNAME(1, "Temperature");
    UA_NodeId parentNodeId = 
        UA_NODEID_NUMERIC(0, UA_NS0ID_OBJECTSFOLDER);
    UA_NodeId parentReferenceNodeId = 
        UA_NODEID_NUMERIC(0, UA_NS0ID_ORGANIZES);
    
    UA_Server_addVariableNode(server, temperatureNodeId, parentNodeId,
                              parentReferenceNodeId, temperatureName,
                              UA_NODEID_NUMERIC(0, UA_NS0ID_BASEDATAVARIABLETYPE),
                              attr, NULL, NULL);
    
    // Run server
    UA_Server_run(server, &running);
    
    UA_Server_delete(server);
    return 0;
}
```

## Key Concepts

- **Master-Slave vs Peer-to-Peer**: Communication paradigms
- **Determinism**: Predictable communication timing
- **Real-Time**: Guaranteed response time
- **Fieldbus**: Industrial network at field level

## Practical Exercises

1. Implement Modbus RTU master on embedded device
2. Create Modbus TCP server for sensor data
3. Analyze Profinet traffic with Wireshark
4. Set up EtherCAT network with multiple slaves
5. Create OPC UA server exposing device data
6. Implement CRC-16 for Modbus
7. Parse Modbus response frames

## Protocol Comparison

| Protocol | Speed | Topology | Determinism | Complexity |
|----------|-------|----------|-------------|------------|
| Modbus RTU | Low | Bus | No | Low |
| Modbus TCP | Medium | Star | No | Low |
| Profibus DP | Medium | Bus | Yes | Medium |
| Profinet | High | Star | Yes | Medium |
| EtherCAT | Very High | Line/Tree | Yes | Medium |
| OPC UA | Medium | Star | No | High |

## Resources

- Modbus Protocol Specification (modbus.org)
- Profibus & Profinet International (PI)
- EtherCAT Technology Group (ethercat.org)
- OPC Foundation (opcfoundation.org)
- Open-source stacks: libmodbus, SOEM, open62541
