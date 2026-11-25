# WebSocket interface

The WebSocket interface is used for all real-time, bidirectional communication. Messages are encoded using [Protocol Buffers](https://protobuf.dev) (proto3).

## Connection

Connect to the WebSocket URL you received during onboarding (`{baseUrl}/ws`, using the `wss://` scheme) with the same Bearer token as for the REST API. Each vehicle requires its own WebSocket connection.

```
GET {baseUrl}/ws  (wss://)
Authorization: Bearer <api_key>
```

TLS 1.3 is used for encryption.

!!! warning "Inactivity timeout"
    If the CSP receives no message on an open WebSocket for 120 seconds, the connection will be closed.

## Why Protocol Buffers?

The dynamic data exchanged over the WebSocket has high-frequency, low-latency requirements. Protocol Buffers provide:

- Compact serialization (reducing bandwidth)
- Cross-platform code generation (C#, C++, Java, JavaScript, Python, and others)
- Extensibility

Use the [protobuf runtime library](https://github.com/protocolbuffers/protobuf?tab=readme-ov-file#protobuf-runtime-installation) for your development environment to parse incoming messages and serialize outgoing messages.

## Message structure

All WebSocket communication uses a single envelope message format (`FmsItsMessage`) containing three fields. Field names on this page are in camelCase, as most generated protobuf code uses them; the [Protobuf specification](protobuf.md) uses the snake_case names from the `.proto` file (for example `vehicle_id` is `vehicle_id`).

| Field | Type | Description |
|---|---|---|
| `message_type` | enum | The type of message |
| `vehicle_id` | string | The vehicle this message relates to. Must be the `id` used in `PUT /vehicles` |
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
| `LOCATION_UPDATE` | `location_update` |
| `TIMESTAMP_MESSAGE` | `timestamp_message` |
| `EVENT_MESSAGE` | `info_container` + `content_container` |
| `ERROR_MESSAGE` | `error_message` |

## Location updates

The FMS party must send location updates for each vehicle over its WebSocket connection. Updates must be sent as a stream (not batched) to support real-time C-ITS services.

**Default rate:** At least 1 update every 5 seconds (0.2 Hz). The traffic light and priority use cases need at least 1 update every second (1 Hz) near an iVRI (the intersection area):

- **With `dynamicRate` enabled (recommended):** the CSP sends the required interval with a `CHANGED_FREQUENCY_REQUEST` event, including when 1 Hz is needed near an iVRI. The requested interval replaces the default rate. See [Dynamic sampling](use-cases.md#dynamic-sampling).
- **Without `dynamicRate`:** send at least 1 update per second while the traffic light or priority use cases are active for the vehicle.

Each location update must include the following fields. Updates without them are discarded:

- `geo_reference` - latitude and longitude, with `type` set to `LAT_LON_POINT_GEO_REFERENCE`
- `heading` - with `unit` set to `DEGREES`
- `speed` - with `unit` set to `KILOMETERS_PER_HOUR`
- `creation_date` - GPS fix timestamp (UTC)

The following fields should also be included:

- `id` - unique ID for this update
- `lifecycle` - set to `ACTUAL`
- `gps_quality` - deviation in meters, 95% probability
- `device_date` - device clock timestamp, useful when the GPS time differs from network time

!!! info "GPS accuracy"
    GPS quality is crucial for accurate event matching. For the iVRI priority use case, the CSP must be able to map coordinates to the planned route. Too much deviation will prevent priority requests.

## Time synchronization

The CSP regularly sends timestamp requests (typically every minute) to verify clock synchronization. The FMS party must respond as quickly as possible.

### Timestamp request (CSP → FMS)

Contains:

- `request_transmission` - timestamp when the CSP sent the request

### Timestamp response (FMS → CSP)

Send the response as a `TIMESTAMP_MESSAGE` with the same `vehicle_id` as the request. It contains:

- `request_transmission` - copied from the request
- `request_reception` - timestamp when the FMS received the request
- `response_transmission` - timestamp when the FMS sends the response

!!! warning
    If the average time difference consistently exceeds 3 seconds, the CSP sends a `TIMEDIFF_TOO_BIG` error message. If no response is received within 3 seconds, the CSP sends a `NO_TIMESTAMP_ANSWER` error message.

## Event messages

The CSP sends ITS event messages containing traffic and infrastructure information relevant to the vehicle's current location and route. Each event message contains:

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

## Connection lifetime and reconnection

Three independent timers apply:

| Timer | Duration | What happens when it expires |
|---|---|---|
| WebSocket inactivity | 120 seconds without a message from the FMS | The WebSocket is closed. Reopen it. |
| Vehicle announcement | 24 hours after the last `PUT /vehicles` | The vehicle expires and its open WebSocket is closed (close code 1003). Send `PUT /vehicles` again, then reopen the WebSocket. |
| Route announcement | 12 hours after `PUT /routes`, or at `endTime` if that is earlier | The route is removed. Send `PUT /routes` again if a route is still needed. |

To avoid interruptions, re-announce the vehicle before its 24 hours expire, also while the WebSocket is open.

If the connection closes for any reason, check whether the vehicle (24 hours) and route (12 hours) announcements are still valid, re-announce them if needed, and reconnect the WebSocket.
