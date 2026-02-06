# 5.2 Wireless & IoT

## Topics Covered

### BLE (Bluetooth Low Energy)

#### Overview
- **Version**: Bluetooth 4.0+ (BLE introduced in 4.0)
- **Purpose**: Low-power wireless communication
- **Range**: 10-100 meters (depending on power class)
- **Data Rate**: 125 kbps to 2 Mbps
- **Frequency**: 2.4 GHz ISM band

#### Architecture

##### Roles
- **Central**: Scanner, initiates connections (e.g., smartphone)
- **Peripheral**: Advertiser, accepts connections (e.g., sensor)
- **Observer**: Listens to advertisements
- **Broadcaster**: Sends advertisements only

##### GAP (Generic Access Profile)
- **Advertising**: Peripheral broadcasts presence
- **Scanning**: Central discovers peripherals
- **Connection**: Establishing link

##### GATT (Generic Attribute Profile)
- **Services**: Collections of characteristics
- **Characteristics**: Data values with properties
- **Descriptors**: Metadata about characteristics

```
Device
└── Service (UUID)
    ├── Characteristic (UUID)
    │   ├── Value
    │   └── Descriptors
    └── Characteristic (UUID)
        ├── Value
        └── Descriptors
```

#### Advertising
```c
// Example advertising packet
struct {
    uint8_t length;     // Length of data
    uint8_t type;       // AD type
    uint8_t data[];     // AD data
} adv_data;

// Common AD types
#define AD_TYPE_FLAGS               0x01
#define AD_TYPE_16BIT_UUID          0x03
#define AD_TYPE_COMPLETE_NAME       0x09
#define AD_TYPE_TX_POWER            0x0A
```

#### GATT Services

##### Standard Services
- **Battery Service (0x180F)**: Battery level
- **Heart Rate Service (0x180D)**: Heart rate measurement
- **Device Information (0x180A)**: Manufacturer, model, etc.

##### Custom Service Example
```c
// Define custom service and characteristic UUIDs
#define CUSTOM_SERVICE_UUID     0x1234
#define TEMPERATURE_CHAR_UUID   0x5678

// GATT server implementation (pseudo-code)
void init_gatt_server(void) {
    // Add service
    uint16_t service_handle;
    ble_gatts_add_service(CUSTOM_SERVICE_UUID, &service_handle);
    
    // Add characteristic
    ble_gatts_char_t char_md = {
        .char_props.read = 1,
        .char_props.notify = 1,
        .uuid = TEMPERATURE_CHAR_UUID,
        .max_len = 2  // 16-bit temperature value
    };
    
    uint16_t char_handle;
    ble_gatts_add_characteristic(service_handle, &char_md, &char_handle);
}

// Update characteristic value
void update_temperature(int16_t temp) {
    uint8_t value[2] = {temp & 0xFF, temp >> 8};
    ble_gatts_update_value(char_handle, value, sizeof(value));
    ble_gatts_notify(char_handle);  // Send notification to subscribed clients
}
```

#### Connection Parameters
- **Connection Interval**: 7.5ms to 4s (how often data exchanged)
- **Slave Latency**: Skip connection events (save power)
- **Supervision Timeout**: Connection considered lost

#### Power Optimization
- **Advertising Interval**: Longer = less power
- **Connection Interval**: Longer = less power
- **Slave Latency**: Allow device to sleep
- **Notifications vs Indications**: Notifications don't require ACK

### Wi-Fi (Embedded Stacks)

#### Overview
- **Standards**: 802.11 b/g/n/ac/ax
- **Frequency**: 2.4 GHz and/or 5 GHz
- **Data Rate**: Up to hundreds of Mbps
- **Power**: Higher than BLE/Zigbee

#### lwIP (Lightweight IP)
- **Purpose**: TCP/IP stack for embedded systems
- **Features**: IPv4/IPv6, TCP, UDP, DHCP, DNS
- **Memory**: Configurable (few KB to hundreds of KB)

##### lwIP Configuration
```c
// lwipopts.h
#define LWIP_DHCP               1
#define LWIP_DNS                1
#define LWIP_TCP                1
#define LWIP_UDP                1
#define LWIP_IPV4               1
#define MEM_SIZE                (10*1024)
#define TCP_MSS                 1460
#define TCP_SND_BUF             (2*TCP_MSS)
```

##### Example: TCP Server
```c
#include "lwip/tcp.h"

static err_t tcp_accept_callback(void *arg, struct tcp_pcb *newpcb, err_t err) {
    // Connection accepted
    tcp_recv(newpcb, tcp_recv_callback);
    return ERR_OK;
}

static err_t tcp_recv_callback(void *arg, struct tcp_pcb *tpcb, 
                               struct pbuf *p, err_t err) {
    if (p != NULL) {
        // Process received data
        tcp_recved(tpcb, p->tot_len);
        
        // Echo back
        tcp_write(tpcb, p->payload, p->len, TCP_WRITE_FLAG_COPY);
        
        pbuf_free(p);
    }
    return ERR_OK;
}

void start_tcp_server(void) {
    struct tcp_pcb *pcb = tcp_new();
    tcp_bind(pcb, IP_ADDR_ANY, 80);
    pcb = tcp_listen(pcb);
    tcp_accept(pcb, tcp_accept_callback);
}
```

#### ESP32/ESP8266 Wi-Fi
```c
#include "esp_wifi.h"
#include "esp_event.h"

void wifi_init_sta(void) {
    esp_netif_init();
    esp_event_loop_create_default();
    esp_netif_create_default_wifi_sta();
    
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    esp_wifi_init(&cfg);
    
    wifi_config_t wifi_config = {
        .sta = {
            .ssid = "MySSID",
            .password = "MyPassword",
        },
    };
    
    esp_wifi_set_mode(WIFI_MODE_STA);
    esp_wifi_set_config(WIFI_IF_STA, &wifi_config);
    esp_wifi_start();
    esp_wifi_connect();
}
```

### MQTT & CoAP (Application Layer)

#### MQTT (Message Queuing Telemetry Transport)

##### Overview
- **Architecture**: Publish-Subscribe
- **Broker**: Central message router
- **Topics**: Hierarchical message routing
- **QoS Levels**: 0 (at most once), 1 (at least once), 2 (exactly once)

##### Topic Structure
```
home/livingroom/temperature
home/bedroom/humidity
factory/machine1/status
```

##### MQTT Client Example
```c
#include "mqtt_client.h"

esp_mqtt_client_handle_t client;

static void mqtt_event_handler(void *handler_args, esp_event_base_t base,
                               int32_t event_id, void *event_data) {
    esp_mqtt_event_handle_t event = event_data;
    
    switch (event->event_id) {
        case MQTT_EVENT_CONNECTED:
            // Subscribe to topic
            esp_mqtt_client_subscribe(client, "sensor/temperature", 0);
            break;
            
        case MQTT_EVENT_DATA:
            // Received message
            printf("Topic: %.*s\n", event->topic_len, event->topic);
            printf("Data: %.*s\n", event->data_len, event->data);
            break;
    }
}

void mqtt_start(void) {
    esp_mqtt_client_config_t mqtt_cfg = {
        .uri = "mqtt://broker.hivemq.com",
        .port = 1883,
    };
    
    client = esp_mqtt_client_init(&mqtt_cfg);
    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, 
                                   mqtt_event_handler, NULL);
    esp_mqtt_client_start(client);
}

void publish_temperature(float temp) {
    char payload[32];
    snprintf(payload, sizeof(payload), "%.2f", temp);
    esp_mqtt_client_publish(client, "sensor/temperature", payload, 0, 1, 0);
}
```

##### MQTT Features
- **Retained Messages**: Last message kept by broker
- **Last Will & Testament**: Message sent if client disconnects unexpectedly
- **Clean Session**: Persistent vs non-persistent session

#### CoAP (Constrained Application Protocol)

##### Overview
- **Purpose**: RESTful protocol for constrained devices
- **Transport**: UDP (unlike MQTT which uses TCP)
- **Design**: Similar to HTTP (GET, POST, PUT, DELETE)
- **Size**: Much smaller than HTTP

##### CoAP vs HTTP
```
HTTP: GET /temperature HTTP/1.1\r\nHost: sensor\r\n\r\n  (~40 bytes)
CoAP: 0x40 0x01 0xXX 0xXX 0xBB 't' 'e' 'm' 'p' ...     (~15 bytes)
```

##### CoAP Message Format
```
[Version][Type][Token Length][Code][Message ID]
  2 bit  2 bit     4 bit       8 bit   16 bit

Type: CON (Confirmable), NON (Non-confirmable), ACK, RST
```

##### CoAP Server Example
```c
#include "coap3/coap.h"

void hnd_get_temperature(coap_resource_t *resource,
                         coap_session_t *session,
                         const coap_pdu_t *request,
                         const coap_string_t *query,
                         coap_pdu_t *response) {
    float temp = read_temperature();
    char buffer[32];
    snprintf(buffer, sizeof(buffer), "%.2f", temp);
    
    coap_add_data_blocked_response(request, response,
                                   COAP_MEDIATYPE_TEXT_PLAIN, 0,
                                   strlen(buffer), (const uint8_t *)buffer);
}

void start_coap_server(void) {
    coap_context_t *ctx = coap_new_context(NULL);
    
    coap_resource_t *resource = 
        coap_resource_init(coap_make_str_const("temperature"), 0);
    coap_register_handler(resource, COAP_REQUEST_GET, hnd_get_temperature);
    coap_add_resource(ctx, resource);
    
    // Event loop
    while (1) {
        coap_io_process(ctx, COAP_IO_WAIT);
    }
}
```

### LoRaWAN

#### Overview
- **LoRa**: Long Range physical layer (Semtech)
- **LoRaWAN**: MAC layer protocol (LoRa Alliance)
- **Range**: 2-5 km (urban), 15+ km (rural)
- **Data Rate**: 0.3 to 50 kbps
- **Frequency**: Regional (868 MHz EU, 915 MHz US, 433 MHz Asia)

#### Architecture
```
End Nodes → Gateways → Network Server → Application Server
```

##### Classes
- **Class A**: Bi-directional, lowest power (TX then RX windows)
- **Class B**: Scheduled receive windows
- **Class C**: Continuous receive, highest power

#### LoRaWAN Message
```c
// Example activation (OTAA - Over The Air Activation)
void lora_init(void) {
    // Device EUI (unique identifier)
    uint8_t dev_eui[8] = {0x00, 0x01, ...};
    
    // Application EUI
    uint8_t app_eui[8] = {0x70, 0xB3, ...};
    
    // Application Key
    uint8_t app_key[16] = {0x2B, 0x7E, ...};
    
    // Join network
    lora_join_otaa(dev_eui, app_eui, app_key);
}

void send_data(uint8_t *data, size_t len) {
    lora_send_uplink(1, data, len, false);  // Port 1, unconfirmed
}
```

### Zigbee

#### Overview
- **Standard**: IEEE 802.15.4 (PHY/MAC) + Zigbee Alliance (upper layers)
- **Frequency**: 2.4 GHz (worldwide), 868 MHz (EU), 915 MHz (US)
- **Range**: 10-100 meters
- **Data Rate**: 250 kbps
- **Topology**: Mesh network

#### Device Types
- **Coordinator**: Network creator (1 per network)
- **Router**: Route messages, extend network
- **End Device**: Low power, doesn't route

#### Zigbee Cluster Library (ZCL)
- **Clusters**: Standard data models
  - On/Off (0x0006)
  - Level Control (0x0008)
  - Temperature Measurement (0x0402)

```c
// Example: Send On/Off command
void zigbee_send_on_cmd(uint16_t dest_addr) {
    zcl_send_command(dest_addr, 
                     ZCL_CLUSTER_ID_GEN_ON_OFF,
                     ZCL_CMD_ON_OFF_ON,
                     NULL, 0);
}
```

## Protocol Comparison

| Protocol | Range | Power | Data Rate | Use Case |
|----------|-------|-------|-----------|----------|
| BLE | 10-100m | Very Low | 2 Mbps | Wearables, beacons |
| Wi-Fi | 50-100m | High | 100+ Mbps | High bandwidth, cloud |
| MQTT | N/A | N/A | N/A | IoT messaging |
| CoAP | N/A | N/A | N/A | Constrained IoT |
| LoRaWAN | 2-15km | Very Low | <50 kbps | Long-range sensors |
| Zigbee | 10-100m | Low | 250 kbps | Home automation mesh |

## Key Concepts

- **Publish-Subscribe**: Decouple sender and receiver
- **Mesh Networking**: Self-healing, multi-hop
- **LPWAN**: Low Power Wide Area Network
- **RESTful**: Representational State Transfer

## Practical Exercises

1. Implement BLE peripheral with custom service
2. Create Wi-Fi connected sensor with MQTT
3. Build CoAP server for resource access
4. Set up LoRaWAN node and gateway
5. Create Zigbee mesh network
6. Compare power consumption of different protocols
7. Implement OTA firmware update over BLE

## Best Practices

1. **Choose protocol based on requirements** (range, power, bandwidth)
2. **Implement security** (encryption, authentication)
3. **Handle reconnection** gracefully
4. **Optimize for power** (sleep modes, intervals)
5. **Use standard profiles** when available
6. **Test in real-world conditions**

## Security Considerations

- **BLE**: Pairing, bonding, LE Secure Connections
- **Wi-Fi**: WPA2/WPA3, certificate-based auth
- **MQTT**: TLS, username/password
- **LoRaWAN**: AES-128 encryption
- **Zigbee**: AES-128, trust center

## Resources

- Bluetooth SIG specifications
- ESP-IDF documentation
- MQTT.org
- RFC 7252 (CoAP)
- LoRa Alliance specifications
- Zigbee Alliance documentation
