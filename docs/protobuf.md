# Protocol Buffers specification

The WebSocket interface uses Protocol Buffers (proto3) for message encoding. The proto specification defines all message types exchanged between FMS and CSP over the WebSocket connection.

Download the specification: [FMS_ITS_Interface_Specification.proto](FMS_ITS_Interface_Specification.proto)

## Envelope message

All communication uses a single envelope message:

```protobuf
message FmsItsMessage {
  MessageType     message_type = 1;
  string          vehicle_id = 2;
  FmsItsContainer container = 3;
}
```

## Container

The container holds the actual message payload. Depending on the `message_type`, different fields are populated:

```protobuf
message FmsItsContainer {
  TimestampMessage    timestamp_message = 1;  // TIMESTAMP_MESSAGE
  InfoContainer       info_container = 2;     // EVENT_MESSAGE
  ContentContainer    content_container = 3;  // EVENT_MESSAGE
  LocationUpdateEvent location_update = 4;    // LOCATION_UPDATE
  ErrorMessage        error_message = 5;      // ERROR_MESSAGE
}
```

## Message types

```protobuf
enum MessageType {
  MESSAGETYPE_UNKNOWN = 0;
  TIMESTAMP_MESSAGE = 1;
  LOCATION_UPDATE = 2;
  EVENT_MESSAGE = 3;
  ERROR_MESSAGE = 4;
}
```

## Location update

Sent from FMS to CSP with current vehicle position information.

```protobuf
message LocationUpdateEvent {
  string                    id = 1;             // Unique ID for this update
  google.protobuf.Timestamp creation_date = 2;  // GPS fix timestamp (UTC)
  GeoReference              geo_reference = 3;  // Lat/lon position
  MeasurementType           speed = 4;          // Vehicle speed
  MeasurementType           heading = 5;        // Vehicle heading
  LifeCycle                 lifecycle = 6;      // Must be ACTUAL
  google.protobuf.Timestamp device_date = 7;    // Device clock timestamp
  float                     gps_quality = 8;    // Deviation in meters (95%)
}
```

The `GeoReference` only supports `LAT_LON_POINT_GEO_REFERENCE`:

```protobuf
message GeoReference {
  double           lat = 1;
  double           lon = 2;
  GeoReferenceType type = 3;
}
```

## Timestamp message

Used for time synchronization between FMS and CSP.

```protobuf
message TimestampMessage {
  google.protobuf.Timestamp request_transmission = 1;   // Set by CSP
  google.protobuf.Timestamp request_reception = 2;      // Set by FMS on receipt
  google.protobuf.Timestamp response_transmission = 3;  // Set by FMS on send
}
```

## Event message

ITS events are sent from CSP to FMS and consist of two containers:

### Info container

General metadata about the event:

```protobuf
message InfoContainer {
  string                    author = 1;           // CSP that created the message
  string                    event_id = 2;         // Unique event ID (unique per CSP)
  MessageType               message_type = 3;     // NEW, UPDATE, or DELETE
  int32                     event_version = 4;    // Version of this event
  google.protobuf.Timestamp creation_time = 5;    // Time of sending (UTC)
  int32                     desired_interval = 6; // Requested GPS sample rate (seconds)
}
```

### Content container

The actual event data:

```protobuf
message ContentContainer {
  ItsEventCode its_event_code = 1;  // Event type identifier
  EventInfo    event_info = 2;      // Event details
}
```

### Event info

```protobuf
message EventInfo {
  string               event_text = 1;              // Textual description
  Location             location = 2;                // Event location (EPSG:4326)
  StaticEventMetadata  static_event_metadata = 3;   // For static events
  DynamicEventMetadata dynamic_event_metadata = 4;  // For dynamic events
}
```

## Static event metadata

Contains specific data depending on the event code. Currently supported fields:

| Field | Applicable event codes |
|---|---|
| `road_configuration` | `LANE_CONFIGURATION` |
| `ivri_container` | `TRAFFIC_LIGHT_STATUS`, `TRAFFIC_LIGHT_TIMING`, `PRIORITY` |
| `lane_info` | `LANE_CONFIGURATION`, `TRAFFIC_LIGHT_STATUS`, `TRAFFIC_LIGHT_TIMING` |

### Road configuration

```protobuf
message RoadConfiguration {
  RoadConfigStatus   road_config_status = 1;    // Lane status
  MeasurementType    max_speed = 2;             // Speed if CHANGED_MAX_SPEED
  RoadConfigLocation road_config_location = 3;  // UPSTREAM or DOWNSTREAM
  bool               flashing = 4;              // Flashing lights present
  bool               red_border = 5;            // Red border present
}
```

Road configuration status values:

| Status | Description |
|---|---|
| `OPEN` | Open (falling green arrow) |
| `CLOSED` / `CLOSED_FOR_ALL_TRAFFIC` | Closed for all traffic (red cross) |
| `MERGE_RIGHT` | Merge right |
| `MERGE_LEFT` | Merge left |
| `END_OF_RESTRICTIONS` | End of restrictions |
| `CLOSED_FOR_YOU` | Closed for your vehicle type |
| `CHANGED_MAX_SPEED` | Temporary speed change |
| `BLANK` | Variable message sign is off |

### iVRI container

```protobuf
message IvriContainer {
  PriorityContainer              priority_container = 1;
  TrafficLightStateInformation   traffic_light_state_information = 2;
  TrafficLightTimingInformation  traffic_light_timing_information = 3;
  TrafficLightSpeedAdvice        traffic_light_speed_advice = 4;
  TrafficLightInformation        traffic_light_info = 5;
}
```

### Priority container

```protobuf
message PriorityContainer {
  string   status = 1;     // Priority request status (see Use cases)
  Maneuver maneuver = 2;   // Direction for which priority is requested
  int32    lane_id = 3;    // Lane for which priority is requested
}
```

### Traffic light states

| State | Description |
|---|---|
| `RED` | Red |
| `AMBER` | Amber |
| `GREEN` | Green |
| `ORANGE_FLASHING` | Orange flashing |
| `DARK` | Dark |
| `NO_LIGHT` | Lane without traffic light |

### Traffic light timing

```protobuf
message TrafficLightTimingInformation {
  google.protobuf.Timestamp min_end_time = 1;   // Earliest possible phase change
  google.protobuf.Timestamp max_end_time = 2;   // Latest possible phase change
  google.protobuf.Timestamp likely_time = 3;    // Most likely phase change time
}
```

### Lane info

```protobuf
message LaneInfo {
  int32             max_lane_id = 1;  // Total number of lanes
  int32             lane_id = 2;      // Lane number (left to right, starting at 1)
  repeated Maneuver maneuvers = 3;    // Possible maneuvers for this lane
}
```

### Maneuver

```protobuf
message Maneuver {
  repeated Direction direction = 1;  // Possible directions: STRAIGHT, LEFT, RIGHT, U_TURN
  bool always_yield = 2;            // Whether to always give priority to intersecting traffic
}
```

## Error message

```protobuf
message ErrorMessage {
  ErrorType     error_type = 1;
  string        error_message = 2;       // Error details
  string        related_field = 3;       // Field that caused the error
  FmsItsMessage related_message = 4;     // Original message
  bytes         related_raw_message = 5; // Raw bytes if parsing failed
}
```

## Measurement units

```protobuf
enum MeasurementUnit {
  UNKNOWN = 0;
  KILOMETERS_PER_HOUR = 10;
  METERS_PER_SECOND = 11;
  KILOGRAM = 20;
  POUND = 21;
  METER = 30;
  CENTIMETER = 31;
  MILLISECOND = 40;
  SECOND = 41;
  MINUTE = 42;
  HOUR = 43;
  DEGREES = 50;  // range [0, 360)
}
```

## ITS event codes

See [Event codes](event-codes.md) for the full list with availability status.
