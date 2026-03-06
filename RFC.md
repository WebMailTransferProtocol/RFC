
# RFC: Web Mail Transport Protocol (WMTP)

**Version:** 1.2
**Date:** 2026-03-07

---

## Changelog

#### Version 1.2 — 2026-03-07

- Clarified that plain WMTP addresses in JSON bodies are transmitted without base64url encoding; only URL query parameters require encoding (Section 4, Section 6.2, Section 6.3)
- Updated all examples to use `alice@www.example.org` as sender and `charlie@www.example.com` as recipient
- Fixed deliver URL to use the Sender Server's address (`www.example.org`) (Section 6.3)
- Added well-known metadata keys `reply`, `thread`, and `format` (Section 5.4)
- Added HTTP Redirect policy: one hop, same registered domain, HTTPS only (Section 7.8)

#### Version 1.1 — 2026-03-06

- Replaced custom encodings with base64url (Section 4)
- Replaced Protocol Buffers with multipart/form-data (Section 5)
- Fixed BCC field semantics (Section 5.2)
- Added attachment filename uniqueness constraint (Section 5.1)
- Added Retry Policy (Section 3.1)
- Added Client Authentication with `password` and `token` methods (Section 7.5)
- Updated HMAC signing to cover attachments in canonical form (Section 7.5)
- Added TLS certificate validation requirement (Section 7.1)
- Added Message Size Limits (Section 7.6)
- Added Rate Limiting and Spam guidance (Section 7.7)
- Added `type` parameter to `?action=version` to distinguish server-to-server and client-to-server contexts (Section 6.1)
- Removed implementation-specific deployment section

#### Version 1.0 — 2026-02-01

Initial draft.

---

## Abstract

The Web Mail Transport Protocol (WMTP) defines how two HTTP servers exchange email-like messages. A server that holds a message notifies the recipient's server, which then fetches the message directly. All communication uses standard HTTP requests and query parameters, requiring no URL rewriting or special server configuration.

---

## 1. Concepts and Roles

**Sender Server:** the server that originates a message and holds it until delivery.

**Recipient Server:** the server that receives delivery notifications and fetches messages on behalf of local users.

A single server may act as both Sender and Recipient.

**WMTP endpoint:** a single script (e.g. `index.php`) installed under a `wmtp/` directory. All protocol actions reach this endpoint via query parameters.

---

## 2. Addressing

### 2.1 Address format

A WMTP address has the form:

```
localpart@server-base
```

`server-base` is the part of the server URL that precedes the `wmtp/` directory. The WMTP endpoint URL is always:

```
https://{server-base}/wmtp/
```

**Examples:**

| Address | WMTP endpoint |
|---|---|
| `alice@www.example.org` | `https://www.example.org/wmtp/` |
| `alice@www.example.org/~alice` | `https://www.example.org/~alice/wmtp/` |

When a server constructs the endpoint URL of another server (e.g. to call `?action=deliver`), it extracts `server-base` as the substring after `@` in the address, then appends `/wmtp/`.

### 2.2 Username syntax

The `localpart` (username) must satisfy the following rules:

- **Allowed characters:** ASCII letters (`a`–`z`, `A`–`Z`), digits (`0`–`9`), hyphen (`-`), underscore (`_`).
- **First and last character:** must be a letter or digit. A single-character username is valid.
- **Case:** case insensitive. Implementations must normalize usernames to lowercase on storage and comparison.

Formal grammar:

```
username     = name-char *( inner-char ) name-char / name-char
name-char    = ALPHA / DIGIT
inner-char   = name-char / "-" / "_"
```

**Valid examples:** `alice`, `bob42`, `first-last`, `a_b_c`, `x`

**Invalid examples:** `-alice` (starts with hyphen), `alice-` (ends with hyphen), `ali ce` (space)

### 2.3 Reserved characters

The characters `+`, `/`, `\`, `=`, `|` are reserved and must not appear in usernames. Reserved for potential future use (e.g. `+` for subaddressing).

---

## 3. Delivery Workflow

```
Sender Server                          Recipient Server
      |                                       |
      |-- GET ?action=version --------------->|  (1) check protocol version
      |<-- 200 { protocol, authentication } --|
      |                                       |
      |-- POST ?action=incoming ------------->|  (2) notify: message ready
      |   body: { from, recipient,            |
      |           messageId, secret }         |  ┐ within the same
      |<-- GET ?action=deliver&… -------------|  | incoming request:
      |--- 200 multipart/form-data ---------->|  | recipient fetches
      |<-- 200 OK ----------------------------|  ┘ then confirms
```

The Sender Server must hold the message before executing this sequence. Local users submit messages via `?action=send` (section 6.4).

1. The Sender Server calls `?action=version` on the Recipient Server to confirm protocol compatibility.
2. The Sender Server calls `?action=incoming` on the Recipient Server to notify it that a message is ready. The notification includes the encoded sender address, recipient address, message ID, and a per-recipient secret.
3. Within the same `incoming` request, the Recipient Server validates the recipient, calls `?action=deliver` on the Sender Server to fetch the message, and stores it locally. The `200 OK` response to `incoming` is sent only after the message is successfully stored. If the fetch fails, the Recipient returns `502`.

### 3.1 Retry Policy

If the delivery sequence fails (network error, `502` from the Recipient Server, or no response), the Sender Server must retry. The following constraints apply:

- The Sender Server must attempt delivery **at least twice per calendar day** for each recipient until either delivery succeeds or seven days have elapsed since the first attempt.
- Implementations may retry more frequently or apply any scheduling strategy (e.g. exponential backoff, time-of-day heuristics), as long as the minimum of two attempts per day is met.
- After seven days without successful delivery, the Sender Server must treat the message as undeliverable, notify the local sender with a bounce notification, and discard the queued message.

The seven-day retry window aligns with the deduplication log described in section 7.4, ensuring that a late successful delivery is never recorded as a duplicate.

---

## 4. Encoding

WMTP uses **base64url** (RFC 4648 §5) to encode values that contain binary data or characters that are unsafe in URLs. The base64url alphabet uses `A`–`Z`, `a`–`z`, `0`–`9`, `-`, `_`, with padding characters (`=`) omitted.

- **Encode:** standard base64 with URL-safe alphabet, omitting trailing `=` padding.
- **Decode:** standard base64url decode; implementations must reject any input containing `+`, `/`, or `=`.

**Example:** `charlie@www.example.com` (recipient address in a URL query parameter) → `Y2hhcmxpZUB3d3cuZXhhbXBsZS5jb20`

Plain ASCII strings such as UUIDs and WMTP addresses in JSON bodies are transmitted without encoding. This specification identifies which fields require base64url encoding in each endpoint.

---

## 5. Message Format

Messages are transmitted as `multipart/form-data` bodies. Each message consists of:

1. A required `data` part (`Content-Type: application/json`) containing the message fields as a JSON object.
2. Zero or more `attachment` parts, each carrying one binary file. Attachment parts follow the `data` part in message order.

### 5.1 Attachment parts

Each attachment is a separate part named `attachment`. The `filename` attribute in the `Content-Disposition` header carries the file name. The `Content-Type` header identifies the file type.

Attachment filenames must be unique within a message. A message containing two or more attachments with the same filename must be rejected with `400 Bad Request`.

**Example attachment part:**

```
Content-Disposition: form-data; name="attachment"; filename="photo.jpg"
Content-Type: image/jpeg

[binary file content]
```

### 5.2 Field semantics

| Field | Required | Description |
|---|---|---|
| `messageId` | yes | UUIDv6 string generated before the first delivery attempt; unchanged across all retries |
| `messageDate` | yes | ISO 8601 date and time set by the Sender Server when it accepts the message from the local client |
| `recipient` | yes | Full WMTP address of the recipient (e.g. `charlie@www.example.com`) |
| `from` | yes | Full WMTP address of the sender |
| `to` | yes | Primary recipient addresses |
| `cc` | no | Carbon-copy addresses |
| `bcc` | no | Blind carbon-copy addresses; always omitted from delivered messages. The Sender Server uses this field internally to determine which recipients receive `incoming` notifications, but never includes it in the delivered message. |
| `subject` | no | Message subject line |
| `content` | no | Message body (plain text or HTML) |
| `metadata` | no | Arbitrary key-value pairs for extensions |

**Validity constraint:** at least one of `subject`, `content`, or at least one attachment part must be present and non-empty. A message lacking all three is rejected with `400 Bad Request`.

### 5.3 Message identifier generation

The client calling `?action=send` generates the `messageId` (UUIDv6) before submitting the message. Client-side generation enables idempotent retries: if the call fails before the server responds, the client retries with the same `messageId`; the server detects and discards the duplicate (see sections 6.4 and 7.4).

WMTP uses UUID version 6 (UUIDv6) as defined in RFC 9562. UUIDv6 reorders the time bits of UUIDv1 for lexicographic sortability without exposing the host MAC address. Identifiers appear in standard lowercase canonical form: `xxxxxxxx-xxxx-6xxx-yxxx-xxxxxxxxxxxx`.

UUIDv7 is an acceptable alternative where UUIDv6 is unavailable.

### 5.4 Well-known metadata keys

The `metadata` field is a free-form key-value map. The following keys are defined by this specification; implementations that support threading should honour them.

| Key | Value | Description |
|---|---|---|
| `reply` | UUIDv6 string | `messageId` of the message this message is a direct reply to |
| `thread` | UUIDv6 string | `messageId` of the root message of the thread |
| `format` | `text/plain` or `text/html` | Content type of the `content` field. Defaults to `text/plain` if absent |

**Threading semantics:** to reply to a message, the client sets `reply` to the parent's `messageId`. It also sets `thread`: if the parent carries a `thread` key, the client copies that value; otherwise it sets `thread` to the same value as `reply`. This lets clients group messages by thread even when intermediate messages are absent.

---

## 6. Endpoints

All endpoints live at the same URL (the WMTP endpoint). The action is selected via the `action` query parameter.

### 6.1 `GET ?action=version`

Returns the protocol version and, when requested by a local client, the supported authentication methods. The optional `type` query parameter specifies the context: `receive` (default) for server-to-server delivery, `send` for local client submission.

**Response body (JSON) — `type=receive` (default):**

```json
{
  "protocol": "1.0"
}
```

**Response body (JSON) — `type=send`:**

```json
{
  "authentication": ["password", "token"],
  "protocol": "1.0"
}
```

**Response codes:**

| Code | Meaning |
|---|---|
| `200 OK` | Success |

---

### 6.2 `POST ?action=incoming`

Notifies the Recipient Server that a message is ready for delivery.

**Request body (JSON):**

| Field | Type | Description |
|---|---|---|
| `from` | string | Sender address |
| `recipient` | string | Recipient address |
| `messageId` | string | Message identifier |
| `secret` | string | Base64url-encoded per-recipient secret (used to authenticate the subsequent deliver request) |

**Example request body:**

```json
{
  "from": "alice@www.example.org",
  "recipient": "charlie@www.example.com",
  "messageId": "1ef5b3e2-1234-6abc-9def-abcdef012345",
  "secret": "YWJjZGVmZ2hpamtsbW5vcA"
}
```

**Response codes:**

| Code | Meaning |
|---|---|
| `200 OK` | Notification accepted; the Recipient Server will fetch the message |
| `400 Bad Request` | Missing or invalid fields |
| `404 Not Found` | Recipient address is not a known local user |
| `502 Bad Gateway` | Recipient Server accepted the notification but failed to fetch the message from the Sender |

**Duplicate detection:** if the `messageId` in the request body is already present in the server's received-message log, the server returns `200 OK` without fetching or storing the message again. See section 7.4.

---

### 6.3 `GET ?action=deliver&messageId=…&secret=…&recipient=…`

Requests the message from the Sender Server. The Recipient Server calls this endpoint after accepting an `incoming` notification.

**Query parameters:**

| Parameter | Description |
|---|---|
| `messageId` | Message identifier (as provided in the `incoming` notification) |
| `secret` | Base64url-encoded per-recipient secret (as provided in the `incoming` notification) |
| `recipient` | Base64url-encoded recipient address (as provided in the `incoming` notification) |

`messageId` is a UUID and contains only URL-safe characters; no encoding is required. `recipient` may contain `@` and `/` and must be base64url-encoded. `secret` contains arbitrary bytes and must be base64url-encoded.

**Example request:**

```
GET https://www.example.org/wmtp/?action=deliver&messageId=1ef5b3e2-1234-6abc-9def-abcdef012345&secret=YWJjZGVmZ2hpamtsbW5vcA&recipient=Y2hhcmxpZUB3d3cuZXhhbXBsZS5jb20
```

**Response:**

On success, the response body is a `multipart/form-data` message as described in section 5. The `data` JSON part contains all message fields except `bcc`. Attachment parts follow.

```
Content-Type: multipart/form-data; boundary=<boundary>
```

**Response codes:**

| Code | Meaning |
|---|---|
| `200 OK` | Message returned successfully |
| `400 Bad Request` | Missing parameters |
| `403 Forbidden` | Secret does not match the recipient, or recipient does not match the message |
| `404 Not Found` | No message exists for the given message ID |

---

### 6.4 `POST ?action=send`

Accepts an outgoing message from a local client and stores it in the outgoing queue for delivery. The request body is `multipart/form-data` as described in section 5.

The `data` JSON part must include an `authentication` object (see section 7.5). The `messageDate` field is set by the server on receipt and must not be included by the client. All other string fields are plain UTF-8.

**Fields in the `data` JSON part:**

| Field | Type | Required | Description |
|---|---|---|---|
| `authentication` | object | yes | Authentication credentials (see section 7.5) |
| `messageId` | string | yes | UUIDv6 identifier generated by the client (see section 5.3) |
| `from` | string | yes | Sender's full WMTP address; must be a known local user |
| `to` | array of strings | yes | Primary recipient addresses |
| `cc` | array of strings | no | Carbon-copy addresses |
| `bcc` | array of strings | no | Blind carbon-copy addresses |
| `subject` | string | no | Message subject |
| `content` | string | no | Message body |
| `metadata` | object | no | Arbitrary key-value pairs |

Binary attachments are submitted as separate `attachment` parts following the `data` part (see section 5.1).

**Validity constraint:** at least one of `subject`, `content`, or at least one attachment part must be present and non-empty (see section 5.2).

**Idempotency:** if `messageId` already exists in the outgoing queue, the server returns `200 OK` without creating a duplicate entry.

**Response codes:**

| Code | Meaning |
|---|---|
| `200 OK` | Message saved and queued for delivery |
| `400 Bad Request` | Missing required fields, malformed body, or message lacks subject, content, and attachments |
| `401 Unauthorized` | Missing or invalid authentication credentials |
| `404 Not Found` | Sender address is not a known local user |

---

## 7. Security

### 7.1 Transport

All WMTP traffic must use HTTPS. Implementations must validate the remote server's TLS certificate against a trusted certificate authority and must reject connections with invalid, expired, or self-signed certificates. Disabling certificate validation is not permitted.

### 7.2 Per-recipient secrets

The Sender Server generates a unique secret for each (message, recipient) pair, includes it in the `incoming` notification, and requires it unchanged in the `deliver` request. This ensures that only the notified Recipient Server can retrieve the message, and only for the specific recipient it was notified about.

Secrets must be generated with a cryptographically secure random number generator. A minimum length of 16 bytes (before encoding) is recommended.

### 7.3 Recipient validation

The Recipient Server must verify that the `recipient` field in an `incoming` notification corresponds to a locally known user before accepting the notification or fetching the message. This prevents the Recipient Server from acting as a relay for arbitrary addresses.

### 7.4 Delivery idempotency

The Recipient Server maintains a log of `(messageId, recipient)` pairs for at least seven days. When `?action=incoming` arrives with a pair already in this log, the server returns `200 OK` without fetching or storing the message again.

Using the pair rather than `messageId` alone is necessary: the same message may be addressed to multiple local users on the same server, each generating a separate `incoming` notification with a different `recipient` value. Deduplicating on `messageId` alone would prevent delivery to the second and subsequent local recipients.

This lets the Sender Server safely retry an `incoming` notification whenever it cannot confirm the previous attempt succeeded. The seven-day window accommodates all practical retry intervals.

### 7.5 Client Authentication

The `?action=send` endpoint requires authentication. A server must implement at least one of the two methods defined below; it may implement both. The `authentication` field in the `?action=version&type=send` response lists the methods supported by that server.

#### Method `password`

The client sends the password of the local user identified by the `from` field:

```json
{
  "method": "password",
  "password": "plaintext-password"
}
```

#### Method `token`

A token consists of two parts: a public identifier (`id`) and a secret known only to the client and the server. The client signs the request using HMAC-SHA256 and includes the result in the `signature` field.

```json
{
  "method": "token",
  "id": "public-token-id",
  "signature": "hmac-sha256-hex",
  "timestamp": 1709730000
}
```

**Signing procedure:** the client constructs the following compact JSON string (fields in alphabetical order, no extra whitespace) and computes `HMAC-SHA256(tokenSecret, signingString)`:

```
{"message":<messageObject>,"timestamp":<timestamp>,"token":"<tokenId>"}
```

Where `<messageObject>` is the compact JSON serialization of the message, constructed as follows:
- All fields from the `data` part except `authentication`, in alphabetical key order.
- An `attachments` array containing one object per attachment, each with two fields in alphabetical order: `{"content":"<base64-encoded bytes>","name":"<filename>"}`. Attachments are sorted by `name` in ascending lexicographic order.

`<timestamp>` is the Unix timestamp integer and `<tokenId>` is the public token identifier. The resulting HMAC is encoded as a lowercase hexadecimal string.

The Content-Type of each attachment is not included in the signature and is not specified by the sender. The Recipient Server determines the file type independently.

**Timestamp validation:** the server rejects any request whose `timestamp` differs from the server's current time by more than 60 seconds.

**Token ownership:** the server must verify that the token identified by `id` belongs to the local user identified by the `from` field. A valid signature from a token owned by a different user must be rejected with `401 Unauthorized`.

**Token lifecycle:** token creation and revocation are left to the implementation. Implementations must support token revocation and may enforce a maximum token lifetime.

### 7.6 Message Size Limits

Implementations must enforce a maximum size on incoming request bodies. Requests that exceed this limit must be rejected with `413 Content Too Large` before processing the body. The specific limit is left to the implementation and should be documented by the server operator.

This applies to both `?action=send` (local client submissions) and `?action=incoming` (notifications from remote Sender Servers). For `?action=send`, the effective limit may be enforced at the web server or runtime level (e.g. Apache `LimitRequestBody`, PHP `post_max_size`) before the WMTP script is reached; clients must treat any `413` response as a permanent rejection of that message.

### 7.7 Rate Limiting and Spam

Deploying a WMTP server may require just a free PHP hosting account. This low barrier is a design goal, but it creates a structural spam risk: a malicious actor can deploy a sender instance, send bulk messages, move to a new hosting account when blocked, and cycle through free providers indefinitely.

The protocol cannot fully solve this problem, but defines the following baseline:

**Recipient Server obligations:**
- Recipient Servers should implement rate limiting on `?action=incoming` requests. Both per-IP and per-domain rate limiting are recommended, as an attacker cycling through hosting providers may change them.
- When a rate limit is exceeded, the server must respond with `429 Too Many Requests`. The response should include a `Retry-After` header indicating when the sender may retry.

**Sender Server obligations:**
- A Sender Server that receives a `429` response must not retry before the time indicated in the `Retry-After` header. If the header is absent, the server must apply its standard retry policy (see section 3.1) and must not retry immediately.

More sophisticated countermeasures — shared blocklists, sender reputation systems, domain-level filtering — are outside the scope of this version and may be addressed in future extensions.

### 7.8 HTTP Redirects

When a WMTP request receives an HTTP redirect (`301`, `302`, `307`, or `308`), the client may follow it. The following constraints apply:

- **One hop only.** The client must not follow a redirect that itself redirects.
- **Same registered domain.** The target must share the registered domain (eTLD+1) with the original URL. `mail.example.org` is a valid target for `example.org`; `otherdomain.com` is not.
- **HTTPS only.** The target must use HTTPS.

A redirect that violates any of these constraints is a delivery failure.

---

## 8. Error Codes Summary

| Code | Used by | Meaning |
|---|---|---|
| `200 OK` | all | Successful operation |
| `400 Bad Request` | all | Missing or invalid parameters |
| `401 Unauthorized` | `send` | Missing or invalid authentication credentials |
| `403 Forbidden` | `deliver` | Secret or recipient mismatch |
| `404 Not Found` | `incoming`, `deliver`, `send` | Unknown recipient (incoming), unknown message (deliver), or unknown local sender (send) |
| `413 Content Too Large` | `send`, `incoming` | Request body exceeds the server's maximum allowed size |
| `429 Too Many Requests` | `incoming` | Rate limit exceeded; the sender must respect the `Retry-After` header if present |
| `502 Bad Gateway` | `incoming` | Recipient Server could not fetch the message from the Sender |

---

## Authors

Stefano Balocco
