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

If priority use cases are desired, route information **must** be provided before the trip starts. For other use cases, providing route information is recommended as it enables us to provide events earlier.

## Regular position flow (position driven)

In position-driven event matching, events are checked each time a location update is received.

```plantuml
@startuml
skinparam sequenceMessageAlign center

participant "FMS" as fms
participant "CSP" as csp

loop every 1-5 seconds
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

The FMS party must provide a location update **at least once every 5 seconds** (0.2 Hz) for each active vehicle. Based on this location, we check for new, updated, or deleted events and send them over the WebSocket.

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

note over fms : Vehicle passes intersection

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
  If > 3 seconds consistently,
  an error message is sent.
end note

@enduml
```

## Unhappy flow: route tracking failed

If a vehicle deviates from its route for too long, priority functionality may fail. The FMS party is notified through a control message.

```plantuml
@startuml
skinparam sequenceMessageAlign center

participant "FMS" as fms
participant "CSP" as csp

fms -> csp : LocationUpdateEvent\n(off route)

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

In case of disconnection, the recovery depends on how long the connection has been down:

| Scenario | Action required |
|---|---|
| Reconnect within 24 hours | Only reconnect the WebSocket |
| Reconnect after 24 hours | Re-submit vehicle and route via REST API, then reconnect WebSocket |
