# WebSocket interface

The WebSocket interface is used for all real-time, bidirectional communication. Messages are encoded using [Protocol Buffers](https://developers.google.com/protocol-buffers) (proto3).

## Connection

Connect to the WebSocket endpoint at `{baseUrl}/ws` using the same Bearer token as for the REST API. Each vehicle requires its own WebSocket connection.

```
wss://{baseUrl}/ws
Authorization: Bearer <api_key>
```

TLS 1.3 is used for encryption.

!!! warning "Inactivity timeout"
    If no activity is detected on an open WebSocket during 5 minutes, the connection will be closed.

## Why Protocol Buffers?

The dynamic data exchanged over the WebSocket has high-frequency, low-latency requirements. Protocol Buffers provide:

- Compact serialization (reducing bandwidth)
- Cross-platform code generation (C#, C++, Java, JavaScript, Python, and others)
- Extensibility

Use the [protobuf runtime library](https://github.com/protocolbuffers/protobuf?tab=readme-ov-file#protobuf-runtime-installation) for your development environment to parse incoming messages and serialize outgoing messages.

## Message structure

All WebSocket communication uses a single envelope message format (`FmsItsMessage`) containing three fields:

| Field | Type | Description |
|---|---|---|
| `messageType` | enum | The type of message |
| `vehicleId` | string | The vehicle this message relates to |
| `container` | object | The message payload |

### Message types

| Type | Direction | Description |
|---|---|---|
| `LOCATION_UPDATE` | FMS → CSP | Vehicle position, heading, and speed |
| `TIMESTAMP_MESSAGE` | Bidirectional | Time synchronization |
| `EVENT_MESSAGE` | CSP → FMS | ITS event information (traffic, priority, etc.) |
| `ERROR_MESSAGE` | CSP → FMS | Error notifications |

### Container contents

The `container` field holds different data depending on the message type:

| Message type | Container field |
|---|---|
| `LOCATION_UPDATE` | `locationUpdateEvent` |
| `TIMESTAMP_MESSAGE` | `timestampMessage` |
| `EVENT_MESSAGE` | `infoContainer` + `contentContainer` |
| `ERROR_MESSAGE` | `errorMessage` |

## Location updates

The FMS party must send location updates for each vehicle over its WebSocket connection. Updates must be sent as a stream (not batched) to support real-time C-ITS services.

**Default rate:** At least 1 update every 5 seconds (0.2 Hz).

When approaching an iVRI, we may request higher frequency updates (up to 1 Hz) via a changed frequency request if `dynamicRate` is enabled.

Each location update should include:

- Latitude and longitude
- Heading (degrees)
- Speed
- GPS timestamp
- Device timestamp (local clock at time of GPS fix)
- GPS quality (deviation in meters, 95% probability)

!!! info "GPS accuracy"
    GPS quality is crucial for accurate event matching. For the iVRI priority use case, we must be able to map coordinates to the planned route. Too much deviation will prevent priority requests.

## Time synchronization

We regularly send timestamp requests (typically every minute) to verify clock synchronization. The FMS party must respond as quickly as possible.

### Timestamp request (CSP → FMS)

Contains:

- `requestTransmission` - timestamp when we sent the request

### Timestamp response (FMS → CSP)

Contains:

- `requestTransmission` - copied from the request
- `requestReception` - timestamp when the FMS received the request
- `responseTransmission` - timestamp when the FMS sends the response

!!! warning
    If the average time difference consistently exceeds 3 seconds, or no response is received within 3 seconds, an error message will be sent.

## Event messages

We send ITS event messages containing traffic and infrastructure information relevant to the vehicle's current location and route. Each event message contains:

- **Info container** - General metadata about the message (not event-specific)
- **Content container** - The actual event data, varying by event type

Currently supported events: `CHANGED_FREQUENCY_REQUEST`, `LANE_CONFIGURATION`, `PRIORITY`, `TRAFFIC_LIGHT_STATUS`, `TRAFFIC_LIGHT_TIMING`, `EMERGENCY_VEHICLE_APPROACHING`. See [Event codes](event-codes.md) for details.

## Error messages

Error messages are sent to the FMS party when issues are detected.

| Error type | Description |
|---|---|
| `ERRORTYPE_UNKNOWN` | Unknown error |
| `MISSING_FIELD` | An expected field is missing |
| `INVALID_FIELD` | A field has an invalid/unsupported value |
| `PRE_CONDITION_NOT_MET` | Required information via the REST API was not provided |
| `DATA_RATE_ERROR` | Location updates not being received according to specification |
| `NO_TIMESTAMP_ANSWER` | Timestamp response not received within expected timeframe |
| `TIMEDIFF_TOO_BIG` | Clock difference between FMS and CSP is too large |
| `INVALID_PROTO` | Protobuf message could not be decoded |

## Reconnection

In case of disconnection:

- **Within 24 hours** of vehicle registration: only reconnect the WebSocket
- **After 24 hours**: re-submit vehicle and route information via the REST API before reconnecting the WebSocket
