# Event codes

Each ITS use case is identified by an `ITSEventCode`. These codes identify the type of event and determine which fields are filled in the Protobuf message.

## Currently supported

The following event codes are currently supported:

| Code | Name | Description |
|---|---|---|
| 001 | `CHANGED_FREQUENCY_REQUEST` | CSP requests the FMS to change the location update frequency to X Hz |
| 104 | `LANE_CONFIGURATION` | Lane configuration changed (variable message signs) |
| 301 | `PRIORITY` | iVRI priority status update |
| 401 | `TRAFFIC_LIGHT_STATUS` | Traffic light status update |
| 402 | `TRAFFIC_LIGHT_TIMING` | Time to green / time to red |
| 209 | `EMERGENCY_VEHICLE_APPROACHING` | Approaching emergency services (early warning) |

In addition to event codes, the following message types are supported:

| Message type | Description |
|---|---|
| `LOCATION_UPDATE` | Vehicle position updates (FMS → CSP) |
| `TIMESTAMP_MESSAGE` | Time synchronization (bidirectional) |
| `ERROR_MESSAGE` | Error notifications (CSP → FMS) |

## Defined in specification (not yet available)

The following event codes are defined in the [Protobuf specification](protobuf.md) but are **not yet supported**. They may become available in a future version.

### Signaling and speed advice

| Code | Name | Description |
|---|---|---|
| 101 | `MAXIMUM_SPEED` | Maximum allowed speed of X km/h |
| 102 | `ADVISORY_SPEED_LIMIT` | Advised speed of X km/h |
| 103 | `OVERTAKING_BAN` | Overtaking prohibited |

### Dangerous situations

| Code | Name | Description |
|---|---|---|
| 201 | `ROAD_WORKS` | Road works ahead |
| 202 | `EVENTS` | Event ahead |
| 203 | `PLACEHOLDER` | Rail level crossings (placeholder) |
| 204 | `TRAFFIC_JAM` | Traffic jam ahead |
| 205 | `STATIONARY_VEHICLE` | Stationary vehicle ahead |
| 206 | `INCIDENT` | Incident ahead |
| 207 | `WEATHER_CONDITIONS` | Bad weather conditions ahead |
| 208 | `OBJECTS_ON_ROAD` | Objects/cargo on the road |
| 210 | `ROAD_INSPECTOR` | Road inspector on the road |
| 211 | `BRIDGE_OPENING` | Open bridge ahead |
| 212 | `GHOST_DRIVER_WARNING` | Ghost driver approaching |

### iVRI

| Code | Name | Description |
|---|---|---|
| 302 | `SELECTIVE_ACCESS` | Selective access systems (placeholder) |
| 403 | `ADVISORY_SPEED` | Traffic light optimal speed advice |
| 404 | `RAIL_CROSSING_STATUS` | Rail level crossing time to open (placeholder) |
| 410 | `TRAFFIC_LIGHT_INFO` | Additional traffic light info |

### Sensor data

| Code | Name | Description |
|---|---|---|
| 501 | `TIRE_PRESSURE_GAUGE` | Tire pressure information |
| 502 | `ALTIMETER` | Vehicle height information |
| 503 | `WEIGHT` | Vehicle weight information |
| 504 | `PARKING_OCCUPATION` | Parking / loading area occupation (placeholder) |
| 505 | `PARKING_RESERVATION` | Parking / loading area reservation (placeholder) |
| 506 | `WILDLIFE_CROSSING` | Animals crossing the road |

### Other

| Code | Name | Description |
|---|---|---|
| 601 | `RESTRICTED_ACCESS` | Zone with restricted access |
