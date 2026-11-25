# Sequence diagrams

## First-time connection

When an FMS connects for the first time, the vehicle information must be registered via the REST API before opening a WebSocket connection.

```plantuml
@startuml
skinparam sequenceMessageAlign center

participant "FMS" as fms
participant "REST API" as rest
participant "WebSocket" as ws

fms -> rest : PUT /vehicles\n(vehicle info)
rest --> fms : 200 OK

fms -> rest : PUT /routes\n(route info, if priority enabled)
rest --> fms : 200 OK

fms -> ws : Open WebSocket connection\n(Bearer token)
ws --> fms : Connection established

fms -> ws : LocationUpdateEvent\n(first position)
ws --> fms : ITS Events\n(if applicable)

@enduml
```

If priority use cases are desired, route information **must** be provided before the trip starts. For other use cases, providing route information is recommended as it enables the CSP to provide events earlier.

## Regular position flow (position driven)

In position-driven event matching, events are checked each time a location update is received.

```plantuml
@startuml
skinparam sequenceMessageAlign center

participant "FMS" as fms
participant "CSP" as csp

loop every 5 seconds (every second near an iVRI for traffic light and priority use cases)
  fms -> csp : LocationUpdateEvent
  csp --> fms : ITS EventMessage(s)\n(if applicable)
end

note over csp
  A ChangedFrequencyRequest
  may be sent if dynamicRate
  is enabled
end note

@enduml
```

The FMS party must provide a location update **at least once every 5 seconds** (0.2 Hz) for each active vehicle. The traffic light and priority use cases need **at least one update per second** (1 Hz) near an iVRI; see [Location updates](websocket.md#location-updates) for how this works with and without `dynamicRate`. Based on this location, the CSP checks for new, updated, or deleted events and sends them over the WebSocket.

## Regular position flow (event driven)

In event-driven matching, events are matched based on the last known or calculated position.

```plantuml
@startuml
skinparam sequenceMessageAlign center

participant "FMS" as fms
participant "CSP" as csp

fms -> csp : LocationUpdateEvent

... event occurs in the area ...

csp -> fms : ITS EventMessage

@enduml
```

## iVRI priority flow

**Precondition:** Route must be registered via `PUT /routes` before the trip starts.

```plantuml
@startuml
skinparam sequenceMessageAlign center

participant "FMS" as fms
participant "CSP" as csp
participant "iVRI" as ivri

fms -> csp : LocationUpdateEvent\n(approaching iVRI)

csp -> ivri : Priority request
ivri --> csp : Status: requested

csp -> fms : EventMessage\n(PRIORITY, status: requested)

ivri --> csp : Status: processing
csp -> fms : EventMessage\n(PRIORITY, status: processing)

ivri --> csp : Status: granted
csp -> fms : EventMessage\n(PRIORITY, status: granted)

note over fms : Vehicle passes stop line

csp -> ivri : Cancel priority request
csp -> fms : EventMessage\n(PRIORITY, status: cancelled)

@enduml
```

## Priority characteristics changed

When vehicle characteristics change (e.g., entering/leaving an area where priority is applicable), the FMS must update the priority state.

```plantuml
@startuml
skinparam sequenceMessageAlign center

participant "FMS" as fms
participant "CSP" as csp

fms -> csp : PUT /vehicles\n(iVRIPriorityEnabled: true)
csp --> fms : 200 OK

note over csp : Priority requests begin\nfor iVRIs on route

... later ...

fms -> csp : PUT /vehicles\n(iVRIPriorityEnabled: false)
csp --> fms : 200 OK

note over csp : Priority requests stop

@enduml
```

## Clock synchronization

```plantuml
@startuml
skinparam sequenceMessageAlign center

participant "FMS" as fms
participant "CSP" as csp

csp -> fms : TimestampMessage\n(requestTransmission)

fms -> csp : TimestampMessage\n(requestTransmission +\nrequestReception +\nresponseTransmission)

note over csp
  Time difference is calculated.
  If > 3 seconds consistently, or no
  response within 3 seconds, an
  error message is sent.
end note

@enduml
```

## Unhappy flow: route tracking failed

When a vehicle approaches an iVRI, the CSP evaluates the priority request against the announced route. If the vehicle's position cannot be matched to that route (for example because the vehicle has deviated from it), no priority is requested and the FMS party is notified through a `PRIORITY` event message with status `noRoute`.

```plantuml
@startuml
skinparam sequenceMessageAlign center

participant "FMS" as fms
participant "CSP" as csp

fms -> csp : LocationUpdateEvent\n(approaching iVRI)

note over csp : Priority evaluation:\nposition does not match\nthe announced route

csp -> fms : EventMessage\n(PRIORITY, status: noRoute)

note over fms
  FMS should:
  1. Recalculate route
  2. PUT new route
  (replaces existing route automatically)
end note

fms -> csp : PUT /routes\n(new route)
csp --> fms : 200 OK

@enduml
```

## Unhappy flow: disconnect

In case of disconnection, the recovery depends on which announcements are still valid (see [Connection lifetime and reconnection](websocket.md#connection-lifetime-and-reconnection)):

| Scenario | Action required |
|---|---|
| Vehicle announced less than 24 hours ago | Only reconnect the WebSocket |
| Vehicle announced 24 hours ago or more | Send `PUT /vehicles`, then reconnect the WebSocket |
| Route announced 12 hours ago or more (or past its `endTime`) | Send `PUT /routes` if a route is still needed |
