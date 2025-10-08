# Frequently Asked Questions (FAQ)

Common questions and answers about the CAN to MQTT Bridge add-on.

## Table of Contents

- [General Questions](#general-questions)
- [Installation & Setup](#installation--setup)
- [CAN Interface](#can-interface)
- [MQTT](#mqtt)
- [Message Format](#message-format)
- [Performance](#performance)
- [Troubleshooting](#troubleshooting)
- [Integration](#integration)

## General Questions

### What is CAN bus?

CAN (Controller Area Network) is a robust vehicle bus standard designed for communication between microcontrollers and devices without a host computer. It's widely used in:

- Automotive systems (OBD-II, engine control, body electronics)
- RVs and campers (RV-C protocol)
- Marine vessels (NMEA 2000)
- Industrial automation (CANopen, DeviceNet)
- Agricultural and construction equipment (J1939)

### What does this add-on do?

The add-on provides a **bidirectional bridge** between CAN bus networks and MQTT:

- **CAN → MQTT**: All received CAN frames are published to MQTT topics
- **MQTT → CAN**: Messages published to MQTT are sent to the CAN bus
- **Automatic initialization**: Sets up CAN interfaces automatically
- **Health monitoring**: Tracks connection status and restarts failed processes

### Do I need special hardware?

Yes, you need CAN interface hardware:

- **USB CAN adapter** (easiest option, works with any HA installation)
- **CAN HAT** for Raspberry Pi (compact, integrated solution)
- **Built-in CAN** on some single-board computers

### What CAN protocols are supported?

The add-on works at the **CAN frame level**, so it supports **all CAN protocols**:

- **OBD-II** (automotive diagnostics)
- **J1939** (heavy vehicles, construction equipment)
- **RV-C** (RV and camper automation)
- **NMEA 2000** (marine navigation)
- **CANopen** (industrial automation)
- **Custom protocols**

You'll need to implement protocol-specific parsing in Home Assistant or Node-RED.

### Is this compatible with all Home Assistant installations?

**Supported platforms:**

- ✅ Home Assistant OS (recommended)
- ❌ Home Assistant Container (limited - requires manual CAN setup)

**Architectures:**

- ✅ amd64 (x86_64)
- ✅ aarch64 (64-bit ARM)
- ✅ armv7 (32-bit ARM)
- ✅ armhf (ARM hard float)
- ✅ i386 (32-bit x86)

## Installation & Setup

### How do I install the add-on?

1. Add the repository URL to Home Assistant:
   ```
   https://github.com/Backroads4Me/can-mqtt-bridge
   ```
2. Install "CAN to MQTT Bridge" from the add-on store
3. Configure CAN interface and MQTT settings
4. Start the add-on

See the [README](../README.md#installation) for detailed steps.

### Do I need to configure MQTT manually?

**Usually no** - the add-on uses **service discovery** to automatically find your MQTT broker (Mosquitto add-on).

**Manual configuration needed if:**

- Using external MQTT broker
- Custom MQTT port or credentials
- SSL/TLS connections

### What MQTT broker should I use?

**Recommended**: Mosquitto add-on (official Home Assistant add-on)

**Also compatible**:

- External Mosquitto broker
- Eclipse Mosquitto
- EMQ X
- HiveMQ
- Any MQTT 3.1.1 compliant broker

### Can I use multiple CAN interfaces?

**Currently**: The add-on supports **one CAN interface** per instance.

## CAN Interface

### What CAN bitrates are supported?

The add-on supports standard CAN bitrates:

- **125 kbps** (125000)
- **250 kbps** (250000) - Common for RV-C and many automotive systems
- **500 kbps** (500000) - Common for OBD-II and automotive
- **1 Mbps** (1000000) - High-speed automotive CAN

**How to choose:**

- Match the bitrate of your CAN network
- Most automotive: 500 kbps
- Most RV-C: 250 kbps
- NMEA 2000: 250 kbps
- J1939: 250 kbps

### My CAN interface is named differently (slcan0, vcan0, etc.)

Change the `can_interface` option to match your interface name:

```yaml
can_interface: "slcan0" # or vcan0, can1, etc.
```

Find your interface name with: `ip link show`

### Can I use CAN FD (Flexible Data-rate)?

**Not currently**. The add-on supports **classic CAN** only (up to 8 data bytes).

CAN FD support may be added in a future release if there's demand.

### How do I test without physical CAN hardware?

Use a **virtual CAN interface** (vcan):

```bash
# Create virtual CAN interface
sudo ip link add dev vcan0 type vcan
sudo ip link set vcan0 up
```

Configure the add-on to use `vcan0` and test with `cansend`/`candump`.

## MQTT

### What MQTT topics does the add-on use?

**Three main topics** (configurable):

- `can/raw` - All received CAN frames (CAN → MQTT)
- `can/send` - Send CAN frames (MQTT → CAN)
- `can/status` - Bridge status (online/offline)

### Can I change the topic names?

Yes, all topics are configurable:

```yaml
mqtt_topic_raw: "my_custom_can/rx"
mqtt_topic_send: "my_custom_can/tx"
mqtt_topic_status: "my_custom_can/status"
```

### What are the QoS and retain settings for MQTT topics?

| Topic        | Direction     | QoS | Retained |
| ------------ | ------------- | --- | -------- |
| `can/raw`    | CAN → MQTT    | 1   | No       |
| `can/send`   | MQTT → CAN    | 1   | No       |
| `can/status` | Add-on → MQTT | 1   | Yes      |

### What message formats are supported?

**CAN Raw Topic (`can/raw`)** - Publishes frames in `ID#DATA` format:
- Standard ID: `123#DEADBEEF` (3 hex digits for ID)
- Extended ID: `18FEF017#0102030405060708` (8 hex digits for ID)

**CAN Send Topic (`can/send`)** - Accepts two formats:
1. **Standard format**: `ID#DATA` (e.g., `123#DEADBEEF`)
2. **Raw hex format**: Continuous hex string (e.g., `19FEDB9406FFFA05FF00FFFF`)
   - First 8 characters = CAN ID
   - Remaining = data bytes
   - Automatically converted to `ID#DATA` format

**CAN Status Topic (`can/status`)** - Simple text messages:
- `bridge_online` - Bridge is running
- `bridge_offline` - Bridge has stopped

### How do I monitor bridge status in Home Assistant?

Create a binary sensor to monitor the bridge:

```yaml
mqtt:
  binary_sensor:
    - name: "CAN Bridge Status"
      state_topic: "can/status"
      payload_on: "bridge_online"
      payload_off: "bridge_offline"
      device_class: connectivity
```
