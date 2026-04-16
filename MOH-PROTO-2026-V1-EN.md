## MoH Proto (Message over HTTPS Protocol)
**RFC-style Minimal Protocol Specification — 2026 / Version 1**

*Document ID:* `MOH-PROTO-2026-V1-EN`  
*Status:* Draft  
*Transport:* HTTPS + Push Notifications

---

## 1. Abstract

MoH Proto (Message over HTTPS Protocol) defines a minimalist, standardized layer on top of HTTPS for bidirectional message exchange between a Client and a Server.

The protocol standardizes:

- Custom HTTP headers (`x-moh-*`)
- A handshake procedure (HTTP `OPTIONS`)
- Interpretation of specific HTTP status codes
- Server-to-client notifications via a gateway URL and/or push messaging

---

## 2. Conventions and Terminology

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are to be interpreted as described in RFC 2119.

- **Client**: A MoH Proto requester (e.g., mobile app, web app).
- **Server**: A MoH Proto responder.
- **Subject (`sub`)**: The identity of a peer, represented by an Ed25519 public key encoded as lowercase hex (32 bytes → 64 hex chars).
- **Signature (`sig`)**: An Ed25519 signature encoded as lowercase hex (64 bytes → 128 hex chars).
- **Timestamp**: A UTC Unix timestamp in milliseconds, represented as a base-10 string.
- **Gateway**: A client-provided HTTPS endpoint used for server-to-client notifications.
- **Root**: A base URL from which relative links are resolved.

Unless explicitly noted, header names are case-insensitive per HTTP, but examples use lowercase.

---

## 3. Transport

- **HTTPS** is used for Client → Server requests and Server → Client direct responses.
- **Push notifications** are used for Server → Client asynchronous delivery. MoH Proto standardizes the notification envelope, not the push-provider-specific transport.

---

## 4. Header Field Registry (x-moh-*)

### 4.1 Common Authentication Headers

Clients and Servers use the following headers to authenticate messages.

- **`x-moh-sub`**: Subject public key, hex-encoded Ed25519 public key.
- **`x-moh-sig`**: Signature, hex-encoded Ed25519 signature.
- **`x-moh-timestamp`**: Timestamp string (Unix ms).

### 4.2 Handshake / Capability Headers

- **`x-moh-notify`** (Client → Server, optional): HTTPS Gateway URL where the Server SHOULD send notifications for this `sub`.
- **`x-moh-root`** (Server → Client): Base URL for resolving relative `Location` values and other relative links.
- **`x-moh-title`** (Server → Client, optional): Display name to show in the Client UI.
- **`x-moh-icon`** (Server → Client, optional): Icon URL (absolute or relative to `x-moh-root`).
- **`x-moh-color`** (Server → Client, optional): UI color hint (e.g., `#RRGGBB`).
- **`x-moh-require`** (Server → Client, optional): Requirement indicator. This spec defines the value `input`.

Clients exposing these headers to application code SHOULD strip the `x-moh-` prefix and present keys in `camelCase` (e.g. `x-moh-root` → `root`).

---

## 5. Cryptographic Profile

### 5.1 Algorithms and Encodings

- **Key type**: Ed25519 public/private keypair.
- **Signature**: Ed25519.
- **Encoding**: lowercase hexadecimal without `0x` prefix.

### 5.2 Canonical Signing String

To reduce ambiguity across implementations, signatures **MUST** be computed over the UTF-8 bytes of the following canonical string:

```
MOH-PROTO/1
<timestamp>
<method>
<request-target>
<notify-url>
<body-sha256-hex>
```

Where:

- `<timestamp>` is the exact value of `x-moh-timestamp`.
- `<method>` is the uppercase HTTP method (e.g. `OPTIONS`, `GET`, `POST`).
- `<request-target>` is the path + optional query, as sent on the wire (e.g. `/status?x=1`).
- `<notify-url>` is the exact value of `x-moh-notify`, or an empty string if absent.
- `<body-sha256-hex>` is the lowercase hex SHA-256 digest of the HTTP message body bytes; for empty bodies this is the SHA-256 of an empty byte array.

### 5.3 Verification Rules

- Recipients **MUST** verify `x-moh-sig` against the signing string and `x-moh-sub`.
- Recipients **MUST** reject messages with timestamps outside an implementation-defined window (RECOMMENDED: ±300 seconds) to mitigate replay.
- Implementations **SHOULD** apply additional replay protections (e.g., nonce tracking) when feasible.

---

## 6. Handshake (Authorization Bootstrap)

### 6.1 Purpose

The handshake lets the Server learn (a) the Client identity (`x-moh-sub`), (b) proof of possession (`x-moh-sig`), and optionally (c) where to deliver notifications (`x-moh-notify`). The Server responds with its own identity and UI metadata.

### 6.2 Request

The Client **MUST** initiate a handshake by sending an HTTP `OPTIONS` request to a Server endpoint (RECOMMENDED: the same endpoint used for subsequent requests).

The Client **MUST** include:

- `x-moh-sub`
- `x-moh-sig`
- `x-moh-timestamp`

The Client **MAY** include:

- `x-moh-notify`

### 6.3 Response

On successful handshake, the Server **MUST** return:

- HTTP `200`
- `x-moh-sub` (Server identity)
- `x-moh-sig`
- `x-moh-timestamp`
- `x-moh-root`

The Server **MAY** return:

- `x-moh-title`
- `x-moh-icon`
- `x-moh-color`

The Server signature **MUST** be computed using the same canonical signing string (Section 5.2) for the response context (method `OPTIONS`, same request-target).

---

## 7. Request/Response Semantics

MoH Proto does not mandate a single request method. Implementations typically use `GET` for reading and `POST` for sending input.

Applications define the meaning of `path` and body payload, but the following status codes are standardized.

---

## 8. Status Codes

### 8.1 401 Unauthorized (Authentication Required)

The Server returns **401** to indicate the Client must provide:

- `x-moh-sub`
- `x-moh-sig`
- `x-moh-timestamp`

Clients receiving 401 **SHOULD** (re)perform the handshake and then retry the original request, if safe.

### 8.2 403 Forbidden with `x-moh-require: input` (Input Required)

The Server returns:

- **403**
- `x-moh-require: input`

to indicate the Client must provide additional input data (application-defined) and re-issue the request.

Clients receiving this response **SHOULD** surface an input UI and then retry with the provided `input`.

### 8.3 301 Moved Permanently with `Location` (Redirect)

The Server returns:

- **301**
- `Location: <uri>`

to request the Client to make another request. If `Location` is relative, the Client **MUST** resolve it against `x-moh-root` (if available) or the original request URL.

### 8.4 200 OK

**200** indicates a normal successful response.

---

## 9. Notifications (Server → Client)

### 9.1 Gateway Notifications (HTTPS)

If the Client provided `x-moh-notify` during the handshake, the Server **SHOULD** use it as a Gateway endpoint for server-to-client notifications.

The Server sends an HTTPS `POST` to the Gateway with:

- `Content-Type: application/json`
- `x-moh-sub` (Server identity)
- `x-moh-sig`
- `x-moh-timestamp`

Body (JSON):

```json
{
  "type": "moh.notify",
  "to": "<client-sub-hex>",
  "path": "/relative-or-absolute-path",
  "body": "optional string payload"
}
```

The notification signature **MUST** be computed using the canonical signing string (Section 5.2) where:

- `<method>` is `POST`
- `<request-target>` is the Gateway request-target
- `<notify-url>` is empty (gateway POST is itself the notification)
- `<body-sha256-hex>` is the SHA-256 of the JSON body bytes

Clients **MUST** verify signatures for gateway notifications.

### 9.2 Push Notifications

For push-provider transports, implementations **MAY** deliver either:

- The full signed JSON envelope (Section 9.1), or
- A minimal push payload that instructs the Client to fetch updates over HTTPS.

This spec standardizes the signed envelope semantics; push-provider-specific fields are out of scope.

---

## 10. Errors

Non-standard errors are application-defined. Servers **SHOULD** include a human-readable body when possible.

---

## 11. Security Considerations

- **Replay attacks**: Implement timestamp validation and a bounded acceptance window; consider nonce tracking per `sub`.
- **Clock skew**: Clients SHOULD keep device time synchronized; servers SHOULD allow a reasonable skew window.
- **Key compromise**: If a Client private key is compromised, the attacker can impersonate the Client. Implementations SHOULD support key rotation (out of scope for V1).
- **Gateway exposure**: A public `x-moh-notify` URL increases attack surface. Gateways MUST enforce signature verification and rate limits.
- **TLS**: All HTTP traffic MUST use HTTPS with modern TLS configuration.

---

## 12. Appendix A: TypeScript-Oriented Model (Non-Normative)

The following illustrates a possible client/server API surface.

```typescript
type MoHProtoRequest = {
  sub: string; // identity of the requester (decoded from hex)
  path: string;
  input?: string; // input data
};

type MoHProtoResponse = {
  sub: string; // identity of the responder (decoded from hex)
  status: "success" | "unauthorized" | "error" | "redirect" | "input";
  body?: string;
  location?: string;
  headers?: Readonly<Record<string, string>>; // x-moh-* headers without prefix, camelCase keys
  raw: Response;
};
```

*End of Specification — Document Ref: MOH-PROTO-2026-V1*

