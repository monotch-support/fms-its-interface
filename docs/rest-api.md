# REST API

The REST API is based on the [Open Trip Model 5.0](https://otm5.opentripmodel.org/) and is used to exchange vehicle and route information. All requests and responses use JSON over HTTPS.

All requests require a Bearer token in the `Authorization` header. See [Authentication](authentication.md) for details.

## Vehicle announcement

Register a vehicle that will connect to the CSP. This must be done **at least every 24 hours** to keep the vehicle active. When the announcement expires, the vehicle's open WebSocket is closed.

The services available for a vehicle are determined by your contract (see [Authorization](authentication.md#authorization)). If your contract enables no services, the announcement is rejected with **403 Forbidden**.

### Create or update vehicle

`PUT /vehicles`

#### Request body

```json
{
  "id": "vehicle-123",
  "name": "Delivery Truck",
  "vehicleType": "truck",
  "licensePlate": "AB-123-CD",
  "fuel": "diesel",
  "externalAttributes": {
    "iVRIPriorityEnabled": false,
    "dynamicRate": false
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Unique identifier for this vehicle |
| `name` | string | No | Display name for the vehicle |
| `vehicleType` | string | Yes | Vehicle type (see [vehicle types](#vehicle-types)) |
| `licensePlate` | string | No | License plate number |
| `fuel` | string | Yes | Fuel type (see [fuel types](#fuel-types)) |
| `externalAttributes` | object | No | Extended properties for this interface |
| `externalAttributes.iVRIPriorityEnabled` | boolean | No | Set to `true` to activate iVRI priority requests for this vehicle |
| `externalAttributes.dynamicRate` | boolean | No | Set to `true` to let the CSP adjust the location update interval dynamically (see [Dynamic sampling](use-cases.md#dynamic-sampling)). Recommended for the traffic light and priority use cases. `false` for the fixed rates (see [Location updates](websocket.md#location-updates)) |

#### Response

**200 OK** - Returns the vehicle object as confirmed by the server.

```json
{
  "id": "vehicle-123",
  "name": "Delivery Truck",
  "vehicleType": "truck",
  "licensePlate": "AB-123-CD",
  "fuel": "diesel",
  "externalAttributes": {
    "iVRIPriorityEnabled": false,
    "dynamicRate": false
  }
}
```

### Get vehicle

`GET /vehicles/{id}`

Returns the current vehicle announcement for the given ID.

**200 OK** - Vehicle found and returned.

**404 Not Found** - Vehicle not found.

### Vehicle types

| Value | Description |
|---|---|
| `lightTruck` | Light truck |
| `boxtruck` | Box truck |
| `truck` | Truck |
| `heavyTruck` | Heavy truck |
| `trailer` | Trailer |
| `specialTransport` | Special transport |
| `dangerousGoods` | Dangerous goods |
| `bus` | Bus |
| `tram` | Tram |
| `pedestrian` | Pedestrian |
| `cyclist` | Cyclist |
| `moped` | Moped |
| `motorcycle` | Motorcycle |
| `car` | Car |
| `passengerCar` | Passenger car |
| `roadWork` | Road work vehicle |
| `rescue` | Rescue vehicle |
| `emergency` | Emergency vehicle |
| `safetyCar` | Safety car |
| `agriculture` | Agricultural vehicle |
| `military` | Military vehicle |
| `roadOperator` | Road operator |
| `taxi` | Taxi |

### Fuel types

| Value |
|---|
| `petrol` |
| `diesel` |
| `electricity` |
| `hydrogen` |
| `lng` |
| `cng` |
| `adBlue` |
| `other` |

---

## Route announcement

Register a route for a vehicle. This is **mandatory** when iVRI priority is enabled (`iVRIPriorityEnabled: true`), and recommended for all other use cases as it enables the CSP to provide route-specific events earlier.

### Create or update route

`PUT /routes`

#### Request body (lat/lon array)

```json
{
  "id": "route-456",
  "name": "Amsterdam to Rotterdam",
  "geoReferences": {
    "type": "latLonArrayGeoReference",
    "points": [
      { "lat": 52.3676, "lon": 4.9041 },
      { "lat": 52.0799, "lon": 4.3113 },
      { "lat": 51.9225, "lon": 4.4792 }
    ]
  },
  "actors": [
    {
      "associationType": "reference",
      "uuid": "vehicle-123",
      "entityType": "vehicle"
    }
  ],
  "actions": [
    {
      "associationType": "inline",
      "entity": {
        "actionType": "genericAction",
        "startTime": "2026-10-19T08:00:00Z",
        "endTime": "2026-10-19T17:00:00Z"
      }
    }
  ]
}
```

#### Request body (address)

```json
{
  "id": "route-789",
  "name": "Delivery to warehouse",
  "geoReferences": {
    "type": "addressGeoReference",
    "name": "Warehouse Rotterdam",
    "street": "Havenstraat",
    "houseNumber": "10",
    "postalCode": "3011 AB",
    "city": "Rotterdam",
    "country": "Netherlands"
  },
  "actors": [
    {
      "associationType": "reference",
      "uuid": "vehicle-123",
      "entityType": "vehicle"
    }
  ]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Unique identifier for this route |
| `name` | string | No | Display name for the route |
| `geoReferences` | object | Yes | Geographic representation (see [geo reference types](#geo-reference-types)) |
| `actors` | array | Yes | Exactly one actor linking the route to a vehicle |
| `actors[].associationType` | string | Yes | Must be `reference` |
| `actors[].uuid` | string | Yes | The vehicle ID this route belongs to |
| `actors[].entityType` | string | Yes | Must be `vehicle` |
| `actions` | array | No | Optional time window for the route |
| `actions[].associationType` | string | If `actions` is given | Must be `inline` |
| `actions[].entity.actionType` | string | If `actions` is given | Must be `genericAction` |
| `actions[].entity.startTime` | string | If `actions` is given | ISO 8601 timestamp. The route becomes active at this time |
| `actions[].entity.endTime` | string | If `actions` is given | ISO 8601 timestamp (must be in the future). `startTime` and `endTime` must be sent together; a request with only one of them is rejected |

!!! info "Route lifecycle"
    - A route expires 12 hours after it was submitted. If `endTime` is earlier, the route expires at `endTime`; a time window cannot extend a route beyond 12 hours
    - With a time window, the route is only active from `startTime`. Before that, no route-specific events or priority requests are provided. Because the 12 hours count from submission, a later `startTime` shortens the active period (for example, `startTime` 2 hours and `endTime` 20 hours after submission gives an active period from 2 to 12 hours after submission)
    - Submitting a new route for a vehicle that already has an active route will automatically replace the existing route
    - Only one active route per vehicle is allowed at any time
    - The vehicle referenced by `actors[].uuid` must be announced first

#### Response

**200 OK** - Returns the route object. Note that the response uses `geoReference` (singular) rather than `geoReferences`.

```json
{
  "id": "route-456",
  "name": "Amsterdam to Rotterdam",
  "creationDate": "2026-10-19T08:30:00Z",
  "lastModified": "2026-10-19T08:30:00Z",
  "geoReference": {
    "type": "latLonArrayGeoReference",
    "points": [
      { "lat": 52.3676, "lon": 4.9041 },
      { "lat": 52.0799, "lon": 4.3113 },
      { "lat": 51.9225, "lon": 4.4792 }
    ]
  },
  "actors": [
    {
      "associationType": "reference",
      "uuid": "vehicle-123",
      "entityType": "vehicle"
    }
  ],
  "actions": [
    {
      "associationType": "inline",
      "entity": {
        "actionType": "genericAction",
        "startTime": "2026-10-19T08:00:00Z",
        "endTime": "2026-10-19T17:00:00Z"
      }
    }
  ]
}
```

### Get route

`GET /routes/{id}`

Returns the current route for the given ID.

**200 OK** - Route found and returned.

**404 Not Found** - Route not found.

### Delete route

`DELETE /routes/{id}`

Removes a route. Returns **200 OK**, also when no route with this ID exists.

!!! tip
    You can explicitly delete a route that is no longer needed, but submitting a new route via `PUT /routes` will also automatically replace any existing active route for the same vehicle.

### Geo reference types

Three geo reference types are supported. `latLonArrayGeoReference` is recommended: a path-based route gives the most accurate route-specific events and priority requests.

#### `latLonArrayGeoReference` (recommended)

An array of latitude/longitude points defining the route polyline. At least two points (start and end) are required. The more detailed the route, the more accurately the CSP can provide route-specific events.

#### `latLonPointGeoReference`

A single latitude/longitude point, used as the destination. Requires `lat` and `lon` fields. The CSP determines the route coordinates to this point.

#### `addressGeoReference`

A single address, used as the destination. The CSP determines the route coordinates to this address.

| Field | Type | Required |
|---|---|---|
| `name` | string | Yes |
| `street` | string | No |
| `houseNumber` | string | No |
| `houseNumberAddition` | string | No |
| `postalCode` | string | No |
| `city` | string | No |
| `country` | string | No |

---

## Error responses

All error responses are returned as JSON:

```json
{
  "code": "invalidField",
  "message": "Invalid value 'xyz'. Allowed values: [truck, bus, ...]"
}
```

Validation errors may return an array of errors:

```json
[
  {
    "code": "missingField",
    "message": "Field 'vehicleType' is required"
  },
  {
    "code": "invalidField",
    "message": "Invalid value 'xyz' for field 'fuel'"
  }
]
```

| HTTP Status | Description |
|---|---|
| 200 | Success |
| 400 | Validation error or precondition not met |
| 401 | Unauthorized - invalid or missing Bearer token |
| 403 | Forbidden - your contract enables no services for this vehicle (see [Authorization](authentication.md#authorization)) |
| 404 | Resource not found |
| 409 | Conflict - another route for the same vehicle was processed at the same moment; resend if needed |
| 500 | Internal server error |

Error codes are case-sensitive; match them exactly as listed.

| Error Code | Description |
|---|---|
| `missingField` | A required field is missing |
| `invalidField` | A field has an invalid value |
| `invalidRequest` | Request-level validation failure |
| `invalidJson` | JSON parsing error |
| `recordNotFound` | Resource does not exist |
| `PRE_CONDITION_NOT_MET` | Business rule violation (e.g., vehicle not announced before route) |
| `concurrentRouteAnnounce` | Another route for the same vehicle was processed at the same moment |
| `UNAUTHORIZED` | Invalid or missing Bearer token |
| `FORBIDDEN` | Your contract enables no services for this vehicle |
