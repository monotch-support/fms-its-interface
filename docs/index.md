# Introduction

The ITS-FMS Interface is a standard for communication between logistics IT-suppliers (FMS) wanting to integrate Intelligent Transport Systems services and ITS Cloud Service Providers (CSP) that offer the required services. These services consist of real-time traffic and infrastructure events that are relevant on the current route of a truck.

The interface is an implementation of the [Open Trip Model 5.0](https://otm5.opentripmodel.org/), an open standard for communication about logistics and traffic. It has been developed together with, and implemented by, several market parties with support from the Dutch government. All certified Cloud Service Providers that have access to the relevant data provide their services following this standard.

The main goal is to make implementation of these services as easy as possible for all software suppliers in the logistics market, to make traffic flow smoother, more predictable and safer for everyone.

## Scope and objective

With the ITS-FMS interface, a set of C-ITS use cases, based on the use cases defined by the Talking Traffic project, are linked to logistics IT-systems (such as Fleet Management Systems), with a specific focus on logistics operators.

The following use cases are **currently supported**:

- Traffic light status and timing information from intelligent traffic light controllers (iVRI)
- Priority allocation at iVRIs
- Lane configuration and variable message sign information
- Early warning messages from approaching emergency vehicles
- Dynamic location update frequency control

Additional use cases are defined in the [Protobuf specification](protobuf.md) and may become available in future versions. See [Event codes](event-codes.md) for the full list.

These are low-latency services delivered over cellular networks.

## Target group

This document is intended for logistics IT-suppliers wanting to integrate ITS services. It is the technical specification for the connection between Fleet Management Systems and Monotch as the ITS Cloud Service Provider (CSP).

## High level architecture

The interface consists of two types of communication:

- **REST API (JSON over HTTPS)** - For static data: vehicle and route information, based on the Open Trip Model
- **WebSocket (Protocol Buffers over WSS)** - For dynamic data: real-time location updates, ITS events, error messages, and time synchronization

There are three implementation options for FMS systems:

1. **Backend-to-backend** - All communication goes through the FMS backend
2. **Direct to vehicle** - Communication goes directly from the on-board unit to the CSP
3. **Hybrid** - REST API via the backend, WebSocket directly from the vehicle

=== "1. Backend-to-backend"

    ```plantuml
    @startuml
    skinparam rectangle {
      BackgroundColor #E8F4FD
      BorderColor #2196F3
    }
    left to right direction

    rectangle "ITS Cloud Service Provider" as csp {
      rectangle "CSP Backend" as backend
    }

    rectangle "FMS" as fms {
      rectangle "FMS Backend" as fmsb
    }

    rectangle "Vehicles / OBU" as vehicle

    backend <-- fmsb : REST API (OTM)\nVehicle & Route
    backend <--> fmsb : WebSocket (Protobuf)\nEvents, location updates,\ncontrol, time sync
    fmsb <--> vehicle : Vehicle data

    @enduml
    ```

=== "2. Direct to vehicle"

    ```plantuml
    @startuml
    skinparam rectangle {
      BackgroundColor #E8F4FD
      BorderColor #2196F3
    }
    left to right direction

    rectangle "ITS Cloud Service Provider" as csp {
      rectangle "CSP Backend" as backend
    }

    rectangle "Vehicles / OBU" as vehicle

    backend <-- vehicle : REST API (OTM)\nVehicle & Route
    backend <--> vehicle : WebSocket (Protobuf)\nEvents, location updates,\ncontrol, time sync

    @enduml
    ```

=== "3. Hybrid"

    ```plantuml
    @startuml
    skinparam rectangle {
      BackgroundColor #E8F4FD
      BorderColor #2196F3
    }
    left to right direction

    rectangle "ITS Cloud Service Provider" as csp {
      rectangle "CSP Backend" as backend
    }

    rectangle "FMS" as fms {
      rectangle "FMS Backend" as fmsb
    }

    rectangle "Vehicles / OBU" as vehicle

    backend <-- fmsb : REST API (OTM)\nVehicle & Route
    backend <--> vehicle : WebSocket (Protobuf)\nEvents, location updates,\ncontrol, time sync
    fmsb <--> vehicle : Vehicle data

    @enduml
    ```

## Interface flow

The following steps describe the typical flow for connecting to the CSP:

1. **Announce vehicle** - Register the vehicle using the REST API (`PUT /vehicles`)
2. **Announce route** - Register the route using the REST API (`PUT /routes`) - required when iVRI priority is enabled
3. **Open WebSocket** - Open one WebSocket connection per vehicle over WSS
4. **Send location updates** - Start communication by sending location updates in Protobuf over the WebSocket
5. **Respond to time sync** - Respond to timestamp requests so the CSP can check clock synchronization, and keep sending location updates to keep the connection open
6. **Receive events** - Receive real-time ITS event messages

!!! warning "Inactivity timeout"
    If the CSP receives no message on an open WebSocket for 120 seconds, the connection will be closed.

## Abbreviations

| Abbreviation | Description |
|---|---|
| API | Application Programming Interface |
| C-ITS | Cooperative Intelligent Transport Systems |
| CSP | (ITS) Cloud Service Provider |
| EPSG | European Petroleum Survey Group (coordinate reference system codes, e.g. EPSG:4326) |
| ETA | Estimated Time of Arrival |
| FMS | Fleet Management System |
| GPS | Global Positioning System |
| HTTPS | Hypertext Transfer Protocol Secure |
| ITS | Intelligent Transport Systems |
| iVRI | Intelligente VerkeersRegelInstallatie (intelligent Traffic Light Controller) |
| JSON | JavaScript Object Notation |
| OBU | On Board Unit |
| OTM | Open Trip Model |
| PBC | Priority Broker Configurator |
| REST | Representational State Transfer |
| SSM | Signal Status Message |
| TLS | Transport Layer Security |
| WSS | WebSocket Secure |

## References

- [Open Trip Model documentation](https://www.opentripmodel.org)
- [CROW standards on C-ITS](https://www.crow.nl/thema-s/verkeersmanagement/landelijke-ivri-standaarden)
- [JSON standards](https://www.json.org)
- [Protocol Buffers](https://protobuf.dev)
