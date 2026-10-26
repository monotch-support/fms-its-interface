# Authentication

## Bearer token

All API requests (REST and WebSocket) must include a Bearer token for authentication. SSL (TLS) is used to encrypt all in-flight data.

The token must be included in the `Authorization` header of every HTTP request:

```http
Authorization: Bearer <api_key>
```

For the WebSocket connection, the same API key is used during the connection handshake.

## Obtaining an API key

An API key will be provided after the onboarding process is complete. Contact [support](mailto:support@monotch.com) to initiate onboarding and receive credentials for both the REST API and WebSocket interfaces.

After onboarding you will receive:

- A unique **API key** (used as Bearer token)
- A **base URL** for the REST API endpoints (`{baseUrl}/vehicles`, `{baseUrl}/routes`)
- A **WebSocket URL** for real-time communication (`{baseUrl}/ws`)

## Authorization

The authorization of FMS parties to subscribe to ITS services is not part of this technical specification. Authorization is described in a separate bilateral contract.
