
# RFC: Web Mail Transport Protocol (WMTP)

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

The `localpart` (username) must satisfy all of the following rules:

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

The characters `+`, `/`, `\`, `=`, `|` are reserved and must not appear in usernames. They are reserved for potential future system use (e.g. `+` for subaddressing).

---

## 3. Delivery Workflow

```
Sender Server                          Recipient Server
      |                                       |
      |-- GET ?action=version --------------->|  (1) check compatibility
      |<-- 200 { supported: { encoding: … }} -|
      |                                       |
      |-- POST ?action=incoming ------------->|  (2) notify: message ready
      |   body: { encoding, from, recipient,  |
      |           messageId, secret, … }      |  ┐ within the same
      |<-- GET ?action=deliver&… -------------|  | incoming request:
      |--- 200 application/protobuf --------->|  | recipient fetches
      |<-- 200 OK ----------------------------|  ┘ then confirms
```

The Sender Server must hold the message before executing this sequence. Local users submit messages via `?action=send` (section 6.4).

1. The Sender Server calls `?action=version` on the Recipient Server to retrieve supported encodings.
2. The Sender Server calls `?action=incoming` on the Recipient Server to notify it that a message is ready. The notification includes the encoded sender address, recipient address, message ID, and a per-recipient secret.
3. Within the same `incoming` request, the Recipient Server validates the recipient, calls `?action=deliver` on the Sender Server to fetch the message, and stores it locally. The `200 OK` response to `incoming` is sent only after the message is successfully stored. If the fetch fails, the Recipient returns `502`.

---

## 4. Encoding

All string fields transmitted in query parameters and JSON bodies must be encoded. Encoding ensures that arbitrary byte sequences (email addresses, message IDs, secrets) survive URL and JSON transport without ambiguity.

WMTP defines two encodings, identified by short string keys.

### 4.1 Encoding `"16"` — Hexadecimal

Each byte is represented as exactly two lowercase hexadecimal characters (`0`–`9`, `a`–`f`).

- **Encode:** `bin2hex( $input )`
- **Decode:** `hex2bin( $input )`

**Example:** `foo@example.com` → `666f6f406578616d706c652e636f6d`

### 4.2 Encoding `"32"` — WMTP Base32

A compact encoding that represents arbitrary bytes using only alphanumeric characters (`0`–`9`, `a`–`z`). It processes input in 5-byte chunks; each chunk becomes exactly 8 output characters.

**Encode algorithm (per 5-byte chunk):**
1. Convert the chunk to a 10-character hexadecimal string.
2. Interpret that string as a base-16 integer.
3. Convert the integer to base 36 (digits `0`–`9` followed by letters `a`–`z`).
4. Left-pad the result with `0` characters to exactly 8 characters.

**Decode algorithm (per 8-character chunk):**
1. Interpret the chunk as a base-36 integer.
2. Convert the integer to hexadecimal.
3. Left-pad the hex string to an even number of characters.
4. Decode the hex string to bytes.

**Example:** `foo@example.com` → `1wogej0o1t6v0s4k0pnqvk0c`

> **Note:** WMTP Base32 is not the same as RFC 4648 Base32. The character set and chunking differ.

### 4.3 Consistency requirement

The encoding declared in a request must be used for all encoded fields within that request. A server must reject any request that mixes encodings or uses an unsupported encoding identifier.

---

## 5. Message Format

Messages are serialized using [Protocol Buffers](https://protobuf.dev/) version 3. The Content-Type for message responses is `application/protobuf`.

### 5.1 Schema

```protobuf
syntax = "proto3";
package wmtp;

message wmail_v1_0 {
    string messageId   = 1;
    string messageDate = 2;
    string recipient   = 3;
    string from        = 4;
    repeated string to  = 5;
    repeated string cc  = 6;
    repeated string bcc = 7;
    optional string subject  = 8;
    optional string content  = 9;
    map<string, string> metadata = 10;

    message attachment {
        string filename = 1;
        bytes  content  = 2;
    }
    repeated attachment attachments = 11;
}
```

### 5.2 Field semantics

| Field | Required | Description |
|---|---|---|
| `messageId` | yes | UUIDv6 string generated before the first delivery attempt; unchanged across all retries |
| `recipient` | yes | Full WMTP address of the recipient (e.g. `bob@example.com`) |
| `from` | yes | Full WMTP address of the sender |
| `to` | yes | Primary recipient addresses |
| `cc` | no | Carbon-copy addresses |
| `bcc` | no | Blind carbon-copy addresses (omit when delivering to non-BCC recipients) |
| `subject` | no | Message subject line |
| `content` | no | Message body (plain text or HTML) |
| `metadata` | no | Arbitrary key-value pairs for extensions |
| `attachments` | no | Binary file attachments |

**Validity constraint:** at least one of `subject`, `content`, or `attachments` must be present and non-empty. A message lacking all three is rejected with `400 Bad Request`.

### 5.3 Message identifier generation

The client calling `?action=send` generates the `messageId` (UUIDv6) before submitting the message. Generating the identifier on the client side enables idempotent retries: if the call fails before the server responds, the client retries with the same `messageId`; the server detects and discards the duplicate (see sections 6.4 and 7.4).

WMTP uses UUID version 6 (UUIDv6) as defined in RFC 9562. UUIDv6 reorders the time bits of UUIDv1 for lexicographic sortability without exposing the host MAC address. Identifiers appear in standard lowercase canonical form: `xxxxxxxx-xxxx-6xxx-yxxx-xxxxxxxxxxxx`.

UUIDv7 is an acceptable alternative where UUIDv6 is unavailable.

---

## 6. Endpoints

All endpoints live at the same URL (the WMTP endpoint). The action is selected via the `action` query parameter.

### 6.1 `GET ?action=version`

Returns the features supported by this server. Clients call this endpoint before sending a delivery notification to confirm encoding compatibility.

**Response body (JSON):**

```json
{
  "supported": {
    "encoding": ["16", "32"]
  }
}
```

**Response codes:**

| Code | Meaning |
|---|---|
| `200 OK` | Success |

---

### 6.2 `POST ?action=incoming`

Notifies the Recipient Server that a message is ready for delivery. All string fields in the JSON body are encoded with the encoding declared in the `encoding` field.

**Request body (JSON):**

| Field | Type | Description |
|---|---|---|
| `encoding` | string | Encoding used for all other fields (`"16"` or `"32"`) |
| `from` | string | Encoded sender address |
| `recipient` | string | Encoded recipient address |
| `messageId` | string | Encoded message identifier |
| `secret` | string | Encoded per-recipient secret (used to authenticate the subsequent deliver request) |
| `supported` | object | Encodings supported by the Sender Server, same format as the version response |

**Example request body (encoding `"16"`):**

```json
{
  "encoding": "16",
  "from": "666f6f406578616d706c652e6f7267",
  "recipient": "62617240777777772e6578616d706c652e636f6d",
  "messageId": "3432",
  "secret": "61626364656667",
  "supported": {
    "encoding": ["16", "32"]
  }
}
```

**Response codes:**

| Code | Meaning |
|---|---|
| `200 OK` | Notification accepted; the Recipient Server will fetch the message |
| `400 Bad Request` | Missing fields, missing or invalid encoding |
| `404 Not Found` | Recipient address is not a known local user |
| `502 Bad Gateway` | Recipient Server accepted the notification but failed to fetch the message from the Sender |

**Duplicate detection:** if the `messageId` in the request body is already present in the server's received-message log, the server returns `200 OK` without fetching or storing the message again. See section 7.4.

---

### 6.3 `GET ?action=deliver&encoding=…&messageId=…&secret=…&recipient=…`

Requests the actual message from the Sender Server. The Recipient Server calls this endpoint after accepting an `incoming` notification. All query parameter values are encoded with the specified encoding.

**Query parameters:**

| Parameter | Description |
|---|---|
| `encoding` | Encoding used for the other parameters (`"16"` or `"32"`) |
| `messageId` | Encoded message identifier (as provided in the `incoming` notification) |
| `secret` | Encoded per-recipient secret (as provided in the `incoming` notification) |
| `recipient` | Encoded recipient address (as provided in the `incoming` notification) |

**Example request:**

```
GET https://example.org/wmtp/?action=deliver&encoding=16&messageId=3432&secret=61626364656667&recipient=62617240777777772e6578616d706c652e636f6d
```

**Response:**

On success, the response body is a Protocol Buffers-serialized `wmail_v1_0` message.

```
Content-Type: application/protobuf
Content-Length: <bytes>
```

**Response codes:**

| Code | Meaning |
|---|---|
| `200 OK` | Message returned successfully |
| `400 Bad Request` | Missing parameters, invalid encoding |
| `403 Forbidden` | Secret does not match the recipient, or recipient does not match the message |
| `404 Not Found` | No message exists for the given message ID |

---

### 6.4 `POST ?action=send`

Accepts an outgoing message from a local client and stores it in the outgoing queue for delivery.

All string fields are plain UTF-8; no WMTP encoding applies to this endpoint.

**Request body (JSON):**

| Field | Type | Required | Description |
|---|---|---|---|
| `messageId` | string | yes | UUIDv6 identifier generated by the client (see section 5.3) |
| `from` | string | yes | Sender's full WMTP address; must be a known local user |
| `to` | array of strings | yes | Primary recipient addresses |
| `cc` | array of strings | no | Carbon-copy addresses |
| `bcc` | array of strings | no | Blind carbon-copy addresses |
| `subject` | string | no | Message subject |
| `content` | string | no | Message body |
| `attachments` | array of objects | no | Each object has `filename` (string) and `content` (base64-encoded bytes) |

**Validity constraint:** at least one of `subject`, `content`, or `attachments` must be present and non-empty (see section 5.2).

**Idempotency:** if `messageId` already exists in the outgoing queue, the server returns `200 OK` without creating a duplicate entry.

**Response codes:**

| Code | Meaning |
|---|---|
| `200 OK` | Message saved and queued for delivery |
| `400 Bad Request` | Missing required fields, malformed body, or message lacks subject, content, and attachments |
| `404 Not Found` | Sender address is not a known local user |

---

## 7. Security

### 7.1 Transport

All WMTP traffic must use HTTPS.

### 7.2 Per-recipient secrets

The Sender Server generates a unique secret for each (message, recipient) pair. The secret is included in the `incoming` notification and must be presented unchanged in the `deliver` request. This ensures that only the notified Recipient Server can retrieve the message, and only for the specific recipient it was notified about.

Secrets must be generated with a cryptographically secure random number generator. A minimum length of 16 bytes (before encoding) is recommended.

### 7.3 Recipient validation

The Recipient Server must verify that the `recipient` field in an `incoming` notification corresponds to a locally known user before accepting the notification or fetching the message. This prevents the Recipient Server from acting as a relay for arbitrary addresses.

### 7.4 Delivery idempotency

The Recipient Server maintains a log of `(messageId, recipient)` pairs for at least seven days. When `?action=incoming` arrives with a pair already in this log, the server returns `200 OK` without fetching or storing the message again.

Using the pair rather than `messageId` alone is necessary: the same message may be addressed to multiple local users on the same server, each generating a separate `incoming` notification with a different `recipient` value. Deduplicating on `messageId` alone would prevent delivery to the second and subsequent local recipients.

This lets the Sender Server safely retry an `incoming` notification whenever it cannot confirm the previous attempt succeeded. The seven-day window accommodates all practical retry intervals.

---

## 8. Error Codes Summary

| Code | Used by | Meaning |
|---|---|---|
| `200 OK` | all | Successful operation |
| `400 Bad Request` | all | Missing or invalid parameters, unsupported or inconsistent encoding |
| `403 Forbidden` | `deliver` | Secret or recipient mismatch |
| `404 Not Found` | `incoming`, `deliver`, `send` | Unknown recipient (incoming), unknown message (deliver), or unknown local sender (send) |
| `502 Bad Gateway` | `incoming` | Recipient Server could not fetch the message from the Sender |

---

## 9. Deployment

A WMTP server is a single script installed under a `wmtp/` directory on any web server. No URL rewriting or special server configuration is required.

The script filename must be the web server's default document (e.g. `index.php` for PHP). All requests reach the script via the query string.

**Minimum deployment steps:**
1. Upload `index.php` to the `wmtp/` directory on any PHP hosting.
2. Configure database credentials in `config.php`.
3. Call `Install()` once to create the database schema.
4. Add at least one local user address to the `users` table.

---

## Authors

Stefano Balocco
