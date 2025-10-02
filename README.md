# SADL (Simple Aviation Datalink) - Specification - 1.0 (Draft)
Created by @Kawba and [GNK Softworks](www.gnksoftworks.com)

## REQUEST FOR COMMENTS: 
This is a draft of the SADL 1.0 Specification that we'd like to use and maintain as an open-source, community-driven standard. This is our first time putting together a specification like this, and before we finalize and implement it, we'd love to hear from those with more experience and strong opinions on where we're falling short and what needs to change. If you'd like to propose a change, please [create an issue](https://github.com/gnk-softworks/SADL/issues/new) in this repository so we can discuss it together.

### Outstanding Points / Discussion Items:

1. **Payload Structure and Encoding**
   - Current spec uses JSON for all message payloads
   - **Benefits**: Developer experience (simplicity of integration and debugging), human-readable, wide library support
   - **Concern**: Verbosity may cause performance issues in high-frequency use cases (e.g., AHRS at 20+ Hz)
   - **Question**: Should we stick with JSON or consider alternatives?
     - Binary format (like GDL90)
     - Efficient binary JSON encodings (CBOR, MessagePack)
     - Protocol Buffers
     - Delimiter-based string format with fixed field positions

### Finalisation Roadmap
- [x] Publish Draft Specification
- [ ] Review / Revision period
- [ ] Finalise Specification
- [ ] Publish Example "Server" application
- [ ] Publish Example "Client" application



## Introduction
SADL (Simple Aviation Datalink) is a free, open-source specification designed to standardize the real-time transmission of situational aviation data between networked devices in flight. The protocol enables seamless sharing of AHRS (attitude and heading), GPS positioning, environmental sensors, and traffic information across diverse systems.

**Key Features:**
- **Simple & Easy to integrate:** JSON-based messaging over WebSockets makes development simple
- **Network Flexible:** Supports wireless, wired, or mixed network topologies
- **Real-time Performance:** Designed for high-frequency updates (20+ Hz for critical data)
- **Extensible:** Modular message types allow for future capability additions
- **Mixed Use:** Designed to be implemented and fit for purpose in real world and simulation settings 

**Primary Use Cases:**
- Connecting instruments (e.g. EFIS) & devices (e.g. ADSB Traffic Receiver) to Electronic Flight Bag (EFB) applications or other instruments
- Streaming flight simulator data to Electronic Flight Bag (EFB) applications or other instruments

### Definitions
| Term | Definition |
| -------- | ------- |
| Server | A device or application that produces and broadcasts situational aviation data using the SADL protocol. Servers announce their presence via UDP broadcast and send data to connected clients via WebSocket. |
| Client | A device or application that discovers and connects to SADL servers to consume situational aviation data. Clients listen for server broadcasts and initiate WebSocket connections. |
| AHRS   | Attitude and Heading Reference System - Real-time pitch, roll, yaw, and heading data derived from MEMS gyroscopes, accelerometers, and magnetometers. |
| EFB | Electronic Flight Bag - A tablet or device running aviation software for flight planning, charts, and situational awareness. |
| WebSocket | A persistent bidirectional communication protocol used for real-time data transmission between SADL servers and clients. |

**Example Use Case:** An AHRS-capable device creates its own WiFi hotspot and broadcasts SADL discovery messages. An EFB application running on a tablet connects to the hotspot, discovers the AHRS device, and establishes a WebSocket connection to receive real-time attitude data.

## Network

The specification has no opinion on if the network is wired, wireless or which device acts as the wireless ap. The specification was imagined with a few setups in mind (examples given below) but is flexible.

Examples:
- Simple - Server creates a wireless network and client devices connect to that network
- Multi Device - A third device category (i.e. ADSB / GDL90 EC device) creates a wireless network and both the server and client devices connect to that wireless network.
- Flight Simulation - A Pseudo Server (A device that reads data from a Flight Simulator and sends using the SADL specification) is connected (Wired or Wireless) to a network. Client devices then connect (wired or wireless) to that network to receive AHRS Data.

## Device Discovery & Connection

SADL uses a client-initiated connection model where clients discover available servers on the network and then establish WebSocket connections to receive data. This approach allows multiple SADL devices to coexist on the same network without broadcasting data to all devices.

**Note on Port Usage:** Both discovery and data transmission use port **5401**. This is intentional and works correctly because discovery uses UDP while the WebSocket connection uses TCP. The two protocols can coexist on the same port number without conflict.

### Discovery Protocol

Servers must broadcast their presence via UDP on port **5401** every **5 seconds** with the following JSON payload. If a server supports multiple SADL specification versions, it must broadcast a separate discovery message for each supported version.

**Discovery Message Fields:**

| Field |	Type |	Required |	Description | Valid Range/Format |
| -------- | ------- | -------- | ------- | ------- |
| device_name |	String |	True |	Human-readable name for the SADL server device (e.g., "GENERIC_AHRS_DEVICE_01"). | 1-64 characters |
| address |	String |	True |	IP address of the server for WebSocket connection (e.g., "192.168.4.124"). | Valid IPv4 address |
| sadl_version |	String |	True |	SADL specification version supported by the server (e.g., "1.0"). | Version string (e.g., "1.0", "1.1") |
| capabilities |	Array of Strings |	True |	List of data types the server can provide. See capabilities table below for valid values. | Array of valid capability strings |
| secure |	Boolean |	True |	Indicates whether the server requires password authentication. Set to `true` if password is required, `false` if open access. | true or false |

**Server Capabilities:**

Capabilities indicate which data types the server can provide. The `HEARTBEAT` message type is automatic and required for all servers, so it should not be included in the capabilities list.

| Capability | Description |
| -------- | ------- |
| AHRS | Server provides attitude and heading data (pitch, roll, slip, rate of turn, heading). |
| GPS | Server provides GPS position data (altitude, speed, track, latitude, longitude). |
| PRESSURE | Server provides pressure altitude and barometric setting data. |
| ENVIRONMENT | Server provides environmental sensor data (CO level, cabin temperature, outside air temperature). |
| TRAFFIC | Server provides traffic target data from ADS-B or similar systems. |

**Example Discovery Broadcast (Open Access):**
```json
{
  "device_name": "GENERIC_AHRS_DEVICE_01",
  "address": "192.168.4.124",
  "sadl_version": "1.0",
  "capabilities": ["AHRS", "GPS", "PRESSURE", "ENVIRONMENT", "TRAFFIC"],
  "secure": false
}
```

**Example Discovery Broadcast (Password Protected):**
```json
{
  "device_name": "GENERIC_AHRS_DEVICE_01",
  "address": "192.168.4.124",
  "sadl_version": "1.0",
  "capabilities": ["AHRS", "GPS", "PRESSURE", "ENVIRONMENT", "TRAFFIC"],
  "secure": true
}
```

### Client Connection Process

1. **Discovery**: Clients should listen for UDP broadcasts on port 5401 and maintain a list of available servers that have broadcast within the last 20 seconds. If a server's `sadl_version` is not supported by the client, that server should not be displayed as an available option to the user.

2. **Device Selection**: Users should be presented with a list of discovered devices (with compatible SADL versions) and select which server to connect to. If the server's `secure` field is `true`, the client must prompt the user to enter a password before attempting connection.

3. **WebSocket Connection**: Once selected, the client initiates a WebSocket connection to:
   ```
   ws://{server_address}:5401/sadl/{sadl_version}/data
   ```
   Example: `ws://192.168.4.124:5401/sadl/1.0/data`

   The WebSocket URL includes the SADL version to enable servers to support multiple specification versions simultaneously.

   **Authentication for Password Protected Servers** (`secure: true`):

   When connecting to a secure server, the client must include the password in the `Authorization` header during the WebSocket handshake using Basic authentication format:
   ```
   Authorization: Basic <base64-encoded-password>
   ```

   The password should be base64-encoded directly (without a username prefix). For example, if the password is `myPassword123`, the header would be:
   ```
   Authorization: Basic bXlQYXNzd29yZDEyMw==
   ```

   **Security Note:** The password is transmitted in plaintext (base64 is encoding, not encryption). This authentication mechanism is only suitable for use on trusted private networks. Care should be taken to ensure networks are properly secured as a primary security mechanism.

   **Connection Responses:**
   - If the password is correct (or not required), the server completes the WebSocket handshake and begins sending data
   - If the password is incorrect or missing when required, the server must reject the WebSocket connection with **401 Unauthorized** status
   - Servers should return a **404 Not Found** response for unsupported versions

4. **Data Reception**: After successful connection, the server begins sending data messages via the WebSocket.

### Connection Termination and Cleanup

**Graceful Disconnection:**
- Either the client or server may initiate a graceful WebSocket disconnect at any time by sending a WebSocket Close frame
- Upon receiving a Close frame, the receiving party should respond with its own Close frame and terminate the connection
- No specific close codes or reasons are required, but standard WebSocket close codes may be used

**Client Responsibilities on Disconnection:**
- Clear all received data (AHRS, GPS, PRESSURE, ENVIRONMENT, TRAFFIC targets) from display
- Stop processing incoming messages
- Return to the discovery state and display the list of available servers
- May attempt automatic reconnection if desired, but should respect user preferences

**Server Responsibilities on Disconnection:**
- Stop sending data messages to the disconnected client
- Free any resources associated with that client connection
- Continue broadcasting discovery messages for other potential clients

**Unexpected Connection Loss:**
- If the WebSocket connection is lost unexpectedly (network failure, timeout, etc.), clients should treat it as a disconnection and follow the client responsibilities above
- Clients may implement automatic reconnection logic with exponential backoff to avoid overwhelming the server
- Servers should detect broken connections using WebSocket ping/pong mechanisms or timeout detection


## Data Messages

After connection the server should send updates via the websocket. There are a number of different message types that should be sent for different data all wrapped by a Message Metadata Object.

**Note:** Data messages sent from server to client do not require acknowledgement from the client. The protocol uses a fire-and-forget model for data transmission to minimize latency.

**Message Ordering and Timestamps:** All message types must include a `timestamp` field in the `content` object. Servers should send messages in chronological order, but clients must track the most recent timestamp and discard messages with older timestamps as a safeguard against out-of-order delivery:
- For `AHRS`, `GPS`, `PRESSURE`, `ENVIRONMENT`, and `HEARTBEAT` messages: Track the most recent timestamp for each message type and discard any message of that type with an older timestamp
- For `TRAFFIC` messages: Track the most recent timestamp per target `uid` and discard any message for that specific target with an older timestamp

### Message Metadata
All data messages sent from the server to client via the websocket connection must be wrapped in a standardized metadata envelope. This envelope provides message routing and identification, allowing clients to efficiently parse and handle different types of data.

**Structure:**
Every message follows this wrapper format:
```json
{
  "message_type": "AHRS",
  "content": {
    //message data here
  }
}
```

**Fields:**

| Field |	Type |	Required |	Description |
| -------- | ------- | -------- | ------- |
| message_type |	String (Enum) |	True |	Identifies the message category. Must be one of: `"AHRS"`, `"GPS"`, `"PRESSURE"`, `"ENVIRONMENT"`, `"TRAFFIC"`, or `"HEARTBEAT"`. Determines the expected structure of the content field. |
| content |	Object  |	True |	Contains the actual data payload specific to the message_type. See individual message type sections below for detailed schema of this object. |



### AHRS Data
AHRS (Attitude and Heading Reference System) data contains attitude and heading information that should be updated as frequently as possible (a minimum of 20 updates per second). Only servers with the `AHRS` capability should send this message type.

| Field |	Type |	Required |	Description |	Valid Range/Format |
| -------- | ------- | -------- | ------- | ------- |
| timestamp |	String |	True |	Message timestamp in ISO 8601 format with milliseconds (e.g., "2025-01-15T14:23:45.123Z"). Must be UTC. Used for message ordering. | ISO 8601 format with milliseconds |
| pitch |	Number |	True |	Current pitch angle in degrees. + is up and - is down. | -90 to +90 |
| roll |	Number  |	True |	Current roll angle in degrees. + is right and - is left. | -180 to +180 |
| slip |	Number  |	False |	Current slip in g. + is right and - is left. | -2.0 to +2.0 |
| rate_of_turn |	Number |	False |	Current rate of turn in degrees per second. + is right and - is left. | -180 to +180 |
| heading |	Number |	False |	Current heading in degrees (0–359). Should be derived using magnetometer. | 0 to 359 |

Example Payload:
```json
{
  "timestamp": "2025-01-15T14:23:45.123Z",
  "pitch": -26.359703,
  "roll": -4.958312,
  "slip": 0.06405502,
  "rate_of_turn": -1.351145,
  "heading": 4
}
```

### GPS Data
GPS data contains position and velocity information from GPS sensors that should be updated as frequently as possible (a minimum of 1 update per second, ideally 5-10 Hz). Only servers with the `GPS` capability should send this message type.

| Field |	Type |	Required |	Description |	Valid Range/Format |
| -------- | ------- | -------- | ------- | ------- |
| timestamp |	String |	True |	Message timestamp in ISO 8601 format with milliseconds (e.g., "2025-01-15T14:23:45.123Z"). Must be UTC. Used for message ordering. | ISO 8601 format with milliseconds |
| latitude |	Number |	True	| Current GPS latitude in decimal degrees. | -90 to +90 |
| longitude |	Number |	True |	Current GPS longitude in decimal degrees. | -180 to +180 |
| alt |	Number |	False |	Current GPS altitude in feet. | -1000 to 100000 |
| speed |	Number | 	False |	Current GPS speed in knots. | 0 to 9999 |
| track |	Number |	False |	Current GPS track in degrees (0–359). | 0 to 359 |

Example Payload:
```json
{
  "timestamp": "2025-01-15T14:23:45.123Z",
  "latitude": 0.000000,
  "longitude": 0.000000,
  "alt": 2499,
  "speed": 146,
  "track": 9
}
```

### Pressure Data
Pressure data contains barometric pressure and derived altitude information that should be updated frequently (a minimum of 1 update per second). Only servers with the `PRESSURE` capability should send this message type.

| Field |	Type |	Required |	Description |	Valid Range/Format |
| -------- | ------- | -------- | ------- | ------- |
| timestamp |	String |	True |	Message timestamp in ISO 8601 format with milliseconds (e.g., "2025-01-15T14:23:45.123Z"). Must be UTC. Used for message ordering. | ISO 8601 format with milliseconds |
| alt |	Number |	True |	Current pressure-derived altitude in feet. | -1000 to 100000 |
| setting |	Number |	False |	Current pressure setting in hPa. Used to calculate pressure.alt. | 900 to 1100 |

Example Payload:
```json
{
  "timestamp": "2025-01-15T14:23:45.123Z",
  "alt": 2499,
  "setting": 1013
}
```

### Environment Data
Contains environmental sensor data that should be updated periodically (approximately once per second). This message type is used for cockpit environment monitoring and outside air conditions. Only servers with the `ENVIRONMENT` capability should send this message type.

| Field |	Type |	Required |	Description | Valid Range/Format |
| -------- | ------- | -------- | ------- | ------- |
| timestamp |	String |	True |	Message timestamp in ISO 8601 format with milliseconds (e.g., "2025-01-15T14:23:45.123Z"). Must be UTC. Used for message ordering. | ISO 8601 format with milliseconds |
| co |	Number |	False |	Current carbon monoxide level in parts per million (ppm). | 0 to 10000 |
| cabin_temp |	Number |	False |	Current cabin temperature in degrees Celsius (°C). | -50 to +70 |
| outside_air_temp |	Number |	False |	Current outside air temperature in degrees Celsius (°C). | -80 to +60 |

Example Payload:
```json
{
  "timestamp": "2025-01-15T14:23:45.123Z",
  "co": 47,
  "cabin_temp": 22,
  "outside_air_temp": 13
}
```

### Heartbeat Data
Heartbeat messages serve as a keep-alive mechanism to maintain the WebSocket connection and confirm the server is still actively transmitting. These messages should be sent once every **30 seconds**. This is particularly useful when other data messages (AHRS, GPS, PRESSURE, ENVIRONMENT, TRAFFIC) may be sent infrequently or not at all.

| Field |	Type |	Required |	Description | Valid Range/Format |
| -------- | ------- | -------- | ------- | ------- |
| timestamp |	String |	True |	Message timestamp in ISO 8601 format with milliseconds (e.g., "2025-01-15T14:23:45.123Z"). Must be UTC. Used for message ordering. | ISO 8601 format with milliseconds |

Example Payload:
```json
{
  "timestamp": "2025-01-15T14:23:45.123Z"
}
```

### Traffic Data
Traffic data represents a single detected aircraft or traffic target. Each traffic message contains position, velocity, and identification information for one target. Multiple traffic messages should be sent as separate messages (each with `message_type: "TRAFFIC"`) as targets are detected or updated. Traffic data should be updated as frequently as new data is available from the source (typically 1-5 Hz depending on the traffic system). Only servers with the `TRAFFIC` capability should send this message type.

#### Traffic Message Fields

| Field |	Type |	Required |	Description | Valid Range/Format |
| -------- | ------- | -------- | ------- | ------- |
| timestamp |	String |	True |	Message timestamp in ISO 8601 format with milliseconds (e.g., "2025-01-15T14:23:45.123Z"). Must be UTC. Used for message ordering per target uid. | ISO 8601 format with milliseconds |
| uid |	String |	True |	Unique identifier for the target. Should be the 24-bit ICAO aircraft address in hexadecimal format (e.g., "A12B3C") if available. Otherwise, use another unique identifier. All updates for the same target must use the same uid. | 1-24 characters, alphanumeric |
| latitude |	Number |	True |	Target latitude in decimal degrees (-90 to +90). | -90 to +90 |
| longitude |	Number |	True |	Target longitude in decimal degrees (-180 to +180). | -180 to +180 |
| altitude |	Number |	True |	Target altitude in feet (pressure altitude or GPS altitude depending on source). | -1000 to 100000 |
| track |	Number |	False |	Target ground track in degrees (0-359). | 0 to 359 |
| ground_speed |	Number |	False |	Target ground speed in knots. | 0 to 9999 |
| vertical_velocity |	Number |	False |	Target vertical speed in feet per minute. Positive is climbing, negative is descending. | -30000 to +30000 |
| callsign |	String |	False |	Aircraft callsign or tail number if available (up to 8 characters). | 1-8 characters, alphanumeric, hyphens, and spaces |
| category |	String (Enum) |	False |	Target category indicating aircraft type. Must be one of: `UNKNOWN`, `ULTRALIGHT`, `LIGHT`, `SMALL`, `LARGE`, `HIGH_VORTEX`, `GLIDER`, `LIGHTER_THAN_AIR`, `SKYDIVER`, `UAV`, `SURFACE_VEHICLE`, `POINT_OBSTACLE`, `OTHER`. | See enum values |

#### Traffic Data Management

**Server Responsibilities:**
- Send regular updates for each tracked target as new data is received from the traffic source
- When a target is no longer tracked (e.g., lost signal, out of range, aged out by the source), simply stop sending updates for that target
- No explicit removal message is required

**Client Responsibilities:**
- Store the latest data for each target using `uid` as the unique key
- Track the timestamp of the last received update for each target
- Automatically remove targets from display if no update has been received after a timeout period
- Make independent decisions on when to generate collision alerts based on position, velocity, and proximity

**Recommended Client Stale Data Timeout:** 5-10 seconds without an update before removing a target from display. Clients may adjust this timeout based on their specific requirements.

Example Payload:
```json
{
  "timestamp": "2025-01-15T14:23:45.123Z",
  "uid": "A4B2C8",
  "latitude": 33.942536,
  "longitude": -118.408075,
  "altitude": 3500,
  "track": 270,
  "ground_speed": 125,
  "vertical_velocity": -500,
  "callsign": "N12345",
  "category": "LIGHT"
}
```


## Commands

In order to allow some interactivity / configuration from the client the server should expect to receive a number of standard commands based on the capabilities of the device. Obviously in some scenarios (i.e. Flight simulation) a few of these commands do not make sense and can be ignored.

### Command Request Format

Commands are sent from client to server over the established WebSocket connection. Each command follows this standard format:

```json
{
    "id": "unique-request-id",
    "command": "COMMAND_NAME",
    "value": "Optional value based on command"
}
```

**Request Fields:**

| Field | Type | Required | Description | Valid Range/Format |
| -------- | ------- | -------- | ------- | ------- |
| id | String | True | Unique identifier for this command request. Used to match responses to requests. Client should generate a unique ID for each command (e.g., UUID, incrementing number, timestamp-based ID). | 1-128 characters, alphanumeric and hyphens |
| command | String | True | The command name to execute. | Valid command name (see command table) |
| value | String/Number | False | Optional value required by specific commands (see command table below). | Depends on command (see command table) |

### Command Response Format

After receiving a command, the server must send a response back to the client over the WebSocket connection. The response follows this format:

```json
{
    "id": "unique-request-id",
    "command": "COMMAND_NAME",
    "status": "SUCCESS",
    "message": "Optional human-readable message"
}
```

**Response Fields:**

| Field | Type | Required | Description | Valid Range/Format |
| -------- | ------- | -------- | ------- | ------- |
| id | String | True | The unique identifier from the original command request. Must exactly match the request id. | Must match request id |
| command | String | True | The command name being acknowledged (must match the request). | Must match request command |
| status | String (Enum) | True | Result of the command. Must be one of: `"SUCCESS"`, `"ERROR"`, `"UNSUPPORTED"`. | SUCCESS, ERROR, or UNSUPPORTED |
| message | String | False | Optional human-readable description of the result or error details. | 0-256 characters |

**Status Values:**
- `SUCCESS`: Command was executed successfully
- `ERROR`: Command failed to execute (e.g., calibration procedure failed, invalid value provided)
- `UNSUPPORTED`: Server does not support this command or it is not applicable in the current context

### Standard Commands

The current standard commands are as follows:

|  Command |  Requires Value | Description | Valid Value Range |
| -------- | ------- | -------- | ------- |
| CALIBRATE_AHRS | No | Runs the calibration procedure of all sensors | N/A |
| LEVEL_AHRS | No | Tell the device to mark the current attitude as level | N/A |
| SET_PRESSURE | Yes | Tells the device to set the value as the pressure setting | 900 to 1100 (hPa) |

**Example Command Exchange:**

Client sends:
```json
{
    "id": "cmd-1234567890",
    "command": "SET_PRESSURE",
    "value": 1013.25
}
```

Server responds:
```json
{
    "id": "cmd-1234567890",
    "command": "SET_PRESSURE",
    "status": "SUCCESS",
    "message": "Pressure setting updated to 1013.25 hPa"
}
```

**Error Example:**

Client sends:
```json
{
    "id": "cmd-1234567891",
    "command": "CALIBRATE_AHRS"
}
```

Server responds:
```json
{
    "id": "cmd-1234567891",
    "command": "CALIBRATE_AHRS",
    "status": "ERROR",
    "message": "Calibration failed: device not stationary"
}
```


## License

The SADL specification is released under the **Creative Commons Attribution 4.0 International (CC BY 4.0)** license, making it free and open for anyone to use, share, and adapt.

**Key Permissions:**
- **Use**: Implement the specification in any aviation software or hardware product
- **Share**: Distribute and publish implementations freely
- **Adapt**: Modify and build upon the specification for any purpose, including commercial use
- **No Royalties**: Free to use with no licensing fees

**Requirements:**
- **Attribution**: Give appropriate credit to the SADL specification and its creators

For the full license text, see the [LICENSE](LICENSE) file in this repository or visit https://creativecommons.org/licenses/by/4.0/

## Contributing

We welcome community feedback and proposals to improve the SADL specification. To contribute:

**Proposal Process:**
1. **Create an Issue**: All proposals, suggestions, and feature requests must be submitted as issues in this GitHub repository
2. **Discussion**: The community and maintainers will discuss the proposal in the issue thread
3. **Review**: If accepted, maintainers will create a linked pull request to incorporate the changes into the next version of the specification
4. **Implementation**: Approved changes will be merged and included in the next specification release

**Important Notes:**
- Independent pull requests submitted directly to this repository will not be accepted
- All changes must originate from a discussed and approved issue
- This process ensures that all specification changes are properly reviewed and documented

To submit a proposal, please [open an issue](https://github.com/gnk-softworks/SADL/issues/new) with a clear description of the suggested change and its rationale.

