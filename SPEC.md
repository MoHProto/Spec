This is a foundational draft for the **MoH Proto Specification v1.0**. It follows the RFC-style structure to ensure technical clarity and authority.

---

# MoH Proto Specification (v1.0-draft)

## 1. Introduction
**MoH (Markdown Over HTTP)** is a minimalist application-level protocol designed for secure, bilateral message exchange between humans, bots, and decentralized agents. It leverages the ubiquity of **HTTP** for transport and the readability of **Markdown** for data representation.

### 1.1 Requirements Language
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

---

## 2. Transport Protocol
MoH **MUST** use HTTP/1.1 or higher (HTTP/2 or HTTP/3 is RECOMMENDED) as its transport layer.

### 2.1 Content Type
All MoH requests and responses containing a body **MUST** use the following Media Type:
`Content-Type: text/markdown; charset=utf-8`

### 2.2 Methods
* **POST**: Used for sending messages or updating state.
* **GET**: Used for retrieving resources or status pages.

---

## 3. Identification and Security
MoH relies on asymmetric cryptography to ensure the authenticity and integrity of messages.

### 3.1 Subject Identification (`X-MoH-Sub`)
The `X-MoH-Sub` header identifies the message sender. It **SHOULD** be a unique identifier (UID) or the Base64/Hex representation of the sender's public key.

### 3.2 Digital Signature (`X-MoH-Sig`)
Every state-changing request (POST/PUT) **MUST** include an `X-MoH-Sig` header.
* **Algorithm**: Ed25519.
* **Encoding**: Base64 or Hex.

### 3.3 Replay Protection (`X-MoH-Timestamp`)
To prevent replay attacks, all signed requests **MUST** include an `X-MoH-Timestamp` (Unix epoch format). Servers **SHOULD** reject requests with a timestamp drift greater than 300 seconds.

### 3.4 Signature Calculation
The signature **MUST** be calculated over a canonical string formed by:
$$Payload = Method + Path + Timestamp + Body$$
*(Note: Body is the raw Markdown string)*.

---

## 4. Notification & Callbacks
MoH supports a reactive "Push" model via the `X-MoH-Notify-URL` header.

### 4.1 Subscription
A client **MAY** include the `X-MoH-Notify-URL` header in any request. The server **MUST** store this URL and associate it with the sender's `sub`.

### 4.2 Callback Execution
When a server-side event occurs, the server **MUST** perform an HTTP POST request to the stored `Notify-URL`.
* The payload **MUST** be a valid Markdown message.
* The callback request **SHOULD** be signed by the server.

---

## 5. Message Structure and Metadata

To maintain the purity of the content, MoH Proto **MUST NOT** use in-body metadata (such as Frontmatter). All service information, styling hints, and protocol instructions **MUST** be transmitted exclusively via HTTP headers prefixed with `X-MoH-*`.

### 5.1 Content Format
By default, all MoH messages use the **Original Markdown (Gruber's Markdown)**.
* **Specification:** [Daring Fireball: Markdown Syntax](https://daringfireball.net/projects/markdown/syntax)

### 5.2 Extended Syntax Negotiation
If a client or server supports extended Markdown features (e.g., GFM, CommonMark), it **MUST** notify the counterparty via the `X-MoH-MD-Version` header.
* **Default (if absent):** `original`
* **Common values:** `gfm`, `commonmark`, `multimarkdown`.

### 5.3 Visual Hints (UI/UX)
MoH supports specific headers to provide a consistent visual identity within client applications. Clients **SHOULD** use these headers to theme the message representation.

* **`X-MoH-Icon-Emoji`**: A single Unicode emoji character representing the service or message context (e.g., `🚀`, `📡`).
* **`X-MoH-Icon-Color`**: The background or accent color for the icon/UI, defined strictly as a **Hexadecimal color code**.
    * **Format**: `^#([A-Fa-f0-9]{6}|[A-Fa-f0-9]{3})$` (e.g., `#FF5733` or `#F00`).
    * **Constraint**: Clients **MUST** reject or ignore non-HEX values to ensure predictable rendering.

---

## 6. Updated Examples

### 6.1 Standard Monitoring (Success)
```http
POST /notify MoH/1.0
Host: gateway.mohproto.org
X-MoH-Sub: status_bot_01
X-MoH-Timestamp: 1713200800
X-MoH-Sig: 7a8b9c...
X-MoH-Icon-Emoji: ✅
X-MoH-Icon-Color: #2ECC71

# System Healthy
All nodes are reporting 100% uptime.
```

### 6.2 Low-Power Alert (IoT Device)
Using a short HEX code for efficiency.

```http
POST /event MoH/1.0
Host: api.mohproto.org
X-MoH-Sub: esp32_sensor_09
X-MoH-Timestamp: 1713200900
X-MoH-Sig: 2d3e4f...
X-MoH-Icon-Emoji: 🔋
X-MoH-Icon-Color: #F00

# Battery Low
Sensor level: 12%
```

---

## 7. Security Considerations
1.  **Transport Security**: All MoH traffic **SHOULD** be encrypted via TLS (HTTPS).
2.  **Key Management**: Clients are responsible for the secure storage of their private keys.
3.  **Privacy**: The `sub` and `Notify-URL` should be treated as sensitive data.
