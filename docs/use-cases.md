# Use cases

This page describes the currently supported use cases in detail.

## Priority at traffic lights

To request priority at a smart traffic light (iVRI), it is **mandatory** that the route of the vehicle is known. We send priority requests in advance to traffic lights along the route.

We may choose to buffer priority responses of downstream traffic lights. In this case, only the latest status of the priority request for the traffic light the vehicle is currently approaching will be sent.

### Priority status flow

The normal flow of priority status values:

1. `requested` - Priority request was detected by the traffic controller
2. `processing` - Request is in queue, other requests are prior
3. `granted` - Prioritization is active
4. `cancelled` - Priority information should no longer be displayed

### Status values from Dutch SSM Profile

| Status | Description |
|---|---|
| `unknown` | Unknown state |
| `requested` | Request detected by the traffic controller |
| `processing` | Checking request (other requests are prior) |
| `watchOtherTraffic` | Cannot give full permission, watch for other traffic |
| `granted` | Prioritization is active |
| `rejected` | Request was rejected by the traffic controller |
| `maxPresence` | Request has exceeded max presence time |
| `reserviceLocked` | Reservice locked, controller requires time before accepting another request |

For latest details, check [CROW](https://www.crow.nl/).

### Additional status values

| Status | Description |
|---|---|
| `cancelled` | After passing the stop line |
| `noAnswer` | No response from iVRI within 1 second |
| `wrongSeqNo` | Wrong response from iVRI (sequence number incorrect) |
| `maxEtaExceeded` | ETA missing or more than 5 minutes in the future |
| `noRoute` | Cannot find matching route |
| `noPbcMatch` | No configuration for route and vehicle |

!!! note
    Failed priority requests do not result in error messages. The reason is communicated through the status field.

## Traffic light status and timing

We send traffic light status and timing information for traffic lights along the vehicle's route.

### Traffic light status

Each message indicates the current state of a traffic light for a specific lane:

| State | Description |
|---|---|
| `RED` | Red |
| `AMBER` | Amber |
| `GREEN` | Green |
| `ORANGE_FLASHING` | Orange flashing |
| `DARK` | Dark |
| `NO_LIGHT` | Lane without traffic light |

### Traffic light timing

Timing information indicates when the traffic light state is expected to change:

- **`min_end_time`** - Earliest possible phase change
- **`max_end_time`** - Latest possible phase change
- **`likely_time`** - Most likely phase change time (between min and max, relevant for traffic-actuated control)

!!! info
    The iVRI container is transmitted when a significant change occurs. Without change, the message is transmitted every 10 seconds.

## Lane configuration

Per lane, an ITS event message with information from the variable message sign is sent. The nearest **upstream** sign (behind the vehicle) defines the current maximum speed and allowed lanes. **Downstream** signs (in front of the vehicle) allow anticipating upcoming changes.

- Variable message sign information is not sent if all signs are blank
- Lane numbers are counted left to right
- The maximum speed value is what is displayed on the variable message sign (not the vehicle-specific maximum speed)

### Road configuration status values

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

## Emergency vehicle approaching

When an emergency vehicle is approaching, we send an `EMERGENCY_VEHICLE_APPROACHING` event. The event includes dynamic metadata with the speed and heading of the emergency vehicle, allowing the FMS to alert the driver in time.

## Dynamic sampling

When `dynamicRate` is enabled on the vehicle, we dynamically adjust the location update frequency by sending `CHANGED_FREQUENCY_REQUEST` events:

- Intervals range between **1 and 60 seconds**
- The FMS party is allowed to send updates more frequently, but updates within 100ms of the last processed update will be discarded
- It is advisable to send an additional location update when the vehicle speed changes significantly (more than 15 km/h since the last update)
