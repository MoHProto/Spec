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

MoH supports both **Authorized** and **Anonymous** interactions. 

### 3.1 Authorization Status
* **Authorized Request**: MUST include both `X-MoH-Sub` and `X-MoH-Sig`.
* **Anonymous Request**: Occurs when `X-MoH-Sub` and/or `X-MoH-Sig` are absent. Servers MAY restrict access to certain resources or actions for anonymous clients.

### 3.2 Subject Identification (`X-MoH-Sub`)
The `X-MoH-Sub` header represents the sender's identity. In MoH, the identity is bound to the cryptographic key pair.
* **Format**: The `X-MoH-Sub` MUST be the **Ed25519 Public Key** encoded as a **Hexadecimal** string.
* **Length**: Exactly 64 characters (representing 32 bytes).
* **Case**: Lowercase Hex is RECOMMENDED.

### 3.3 Key Generation & Identity Creation
To create a MoH identity, a client must:
1.  **Generate an Ed25519 key pair**: Using a secure entropy source.
2.  **Derive the Public Key**: Extract the 32-byte raw public key.
3.  **Encode to Hex**: Convert the bytes to a 64-character string. This string becomes the permanent `X-MoH-Sub`.

> **Implementation Example (JavaScript):**
> ```javascript
> import * as ed from '@noble/ed25519';
> const privateKey = ed.utils.randomPrivateKey();
> const publicKey = await ed.getPublicKey(privateKey);
> const sub = Buffer.from(publicKey).toString('hex'); 
> // Result: "f3b2...a1e0" (64 chars)
> ```

---

## 4. Digital Signature (`X-MoH-Sig`)

For authorized requests, the signature ensures that the message hasn't been tampered with and was sent by the owner of the `X-MoH-Sub`.

### 4.1 Algorithm
* **Type**: Ed25519.
* **Encoding**: Hexadecimal (128 characters representing 64 bytes).

### 4.2 Signature Calculation
The signature MUST be calculated over a UTF-8 string formed by the concatenation of:
$$Payload = Method + Path + Timestamp + Body$$

* **Method**: Uppercase (e.g., `POST`).
* **Path**: The full request path (e.g., `/status`).
* **Timestamp**: The value of `X-MoH-Timestamp`.
* **Body**: The raw, unparsed Markdown string.

---

## 5. Notification & Callbacks
MoH supports a reactive "Push" model via the `X-MoH-Notify-URL` header.

### 5.1 Subscription
A client **MAY** include the `X-MoH-Notify-URL` header in any request. The server **MUST** store this URL and associate it with the sender's `sub`.

### 5.2 Callback Execution
When a server-side event occurs, the server **MUST** perform an HTTP POST request to the stored `Notify-URL`.
* The payload **MUST** be a valid Markdown message.
* The callback request **SHOULD** be signed by the server.

---

## 6. Interaction Examples

### 6.1 Authorized Request (Full Security)
```http
POST /update MoH/1.0
Host: api.mohproto.org
Content-Type: text/markdown
X-MoH-Sub: 464f5245564552...[64 chars]
X-MoH-Timestamp: 1713200000
X-MoH-Sig: 58fb2...[128 chars]

# Status update
Authorized content.
```

### 6.2 Anonymous Request (No Sub/Sig)
```http
GET /public-info MoH/1.0
Host: api.mohproto.org

# Welcome
This is a public page accessible without authorization.
```

### 6.3 Partial/Failed Authorization
If a client provides `X-MoH-Sub` but the `X-MoH-Sig` is missing or invalid, the server **MUST** treat the request as unauthorized and **SHOULD** return an `HTTP 401 Unauthorized` response.

---

## 7. Message Structure and Metadata

To maintain the purity of the content, MoH Proto **MUST NOT** use in-body metadata (such as Frontmatter). All service information, styling hints, and protocol instructions **MUST** be transmitted exclusively via HTTP headers prefixed with `X-MoH-*`.

### 7.1 Content Format
By default, all MoH messages use the **Original Markdown (Gruber's Markdown)**.
* **Specification:** [Daring Fireball: Markdown Syntax](https://daringfireball.net/projects/markdown/syntax)

### 7.2 Extended Syntax Negotiation
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

## 8. Updated Examples

### 8.1 Standard Monitoring (Success)
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

### 8.2 Low-Power Alert (IoT Device)
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

## 9. Security Considerations
1.  **Transport Security**: All MoH traffic **SHOULD** be encrypted via TLS (HTTPS).
2.  **Key Management**: Clients are responsible for the secure storage of their private keys.
3.  **Privacy**: The `sub` and `Notify-URL` should be treated as sensitive data.
