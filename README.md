## 1. Overview

OpenHerd is a decentralized, anonymous communication platform based on a FOSS P2P architecture. This document outlines the technical specifications for post creation, message signing, and data transmission.

## 2. Basic Envelope

Each post or reply MUST generate a new PGP key, which serves as its unique identifier.

### Envelope Structure

```json
{
  "signature": "-----BEGIN PGP SIGNATURE-----...",
  "publicKey": "-----BEGIN PGP PUBLIC KEY BLOCK-----...",
  "id": "keyID",
  "data": "{\"id\":\"keyID\",\"text\":\"...\",\"latitude\":...,\"longitude\":...,\"date\":\"...\",\"parent\":\"...\"}"
}
```
- `signature`: Detached OpenPGP signature of the `data` field.
- `publicKey`: The OpenPGP public key for this post.
- `id`: The fingerprint of the public key (used as the post ID).
- `data`: Stringified JSON containing the post content.

## 3. Creating a New Post

- A new PGP key is generated for each post.
- The post content is signed using the private key.
- Location data is included as-is (consider randomizing for anonymity at the application layer).
- Posts are sent via the `posts` Gossipsub topic.

### Parent-Child Structure

Each post may reference its parent by using the parent’s PGP key ID (`parent`).  
Example:  
- `null` (root post) → `8558e99c353bbac709e470b6342241c315fe352a` (reply) → `6a63e1d077d496691a6047a35ed0d5d1a959fb40` (nested reply)

### Post Format (inside `data`)

```json
{
  "id": "keyID",
  "text": "Hello world!",
  "latitude": 33.7501,
  "longitude": -84.3885,
  "date": "2025-01-30T07:26:06.506Z",
  "parent": "8558e99c353bbac709e470b6342241c315fe352a"
}
```
- `parent` is optional and omitted for root posts.

## 4. Catchup

Clients can request past posts by sending a message to the `catchup` topic.

### Request Format

```json
{}
```
or with an optional `max` parameter:
```json
{
  "max": 200
}
```
- `max` (optional): Maximum number of posts to retrieve (default: 100, minimum: 1).

### Response

- Responses are sent to the `backlog` topic.
- The response is an array of post envelopes (see section 2), chunked if necessary.

## 5. Chunking

Messages (including posts and catchup responses) are chunked to ensure reliable transmission.

### Chunk Format

```json
{
  "index": 0,
  "total": 6,
  "content": "string up to 500 characters"
}
```
- `index`: 0-based index of this chunk.
- `total`: Total number of chunks in the message.
- `content`: String content of this chunk (≤ 500 characters).

### Guidelines

- Chunks must be reassembled in order upon receipt.
- All chunked messages use this format for transmission over Gossipsub topics.

## 6. HTTP Sync Endpoints

These optional HTTP endpoints allow for post synchronization between nodes or lightweight gateway servers. All endpoints are prefixed with `/_openherd`.

### `/outbox` (GET)

Returns all locally stored posts in envelope format.

**Example Response:**

```json
[
  {
    "signature": "-----BEGIN PGP SIGNATURE-----\n...-----END PGP SIGNATURE-----\n",
    "publicKey": "-----BEGIN PGP PUBLIC KEY BLOCK-----\n...-----END PGP PUBLIC KEY BLOCK-----\n",
    "id": "2fef8ec4334abede9aeb1d40293f2d6dbcc1edd0",
    "data": "{\"id\":\"2fef8ec4334abede9aeb1d40293f2d6dbcc1edd0\",\"text\":\"test\",\"latitude\":\"33.5583\",\"longitude\":\"-84.2541\",\"date\":\"2025-06-03T02:06:56.465Z\"}"
  }
]
```

### `/inbox` (POST)

Accepts an array of post envelopes and attempts to import them into the local store.

**Request Body:**

```json
[
  {
    "signature": "-----BEGIN PGP SIGNATURE-----\n...-----END PGP SIGNATURE-----\n",
    "publicKey": "-----BEGIN PGP PUBLIC KEY BLOCK-----\n...-----END PGP PUBLIC KEY BLOCK-----\n",
    "id": "2fef8ec4334abede9aeb1d40293f2d6dbcc1edd0",
    "data": "{\"id\":\"2fef8ec4334abede9aeb1d40293f2d6dbcc1edd0\",\"text\":\"test\",\"latitude\":\"33.5583\",\"longitude\":\"-84.2541\",\"date\":\"2025-06-03T02:06:56.465Z\"}"
  }
]
```

**Response:**

```json
{ "ok": true }
```

* If the body is not an array, responds with `400 Bad Request`.

### `/sync` (POST)

Attempts to synchronize with another OpenHerd node via HTTP or libp2p multiaddr.

**Request Body:**

```js
{
  "address": "http://example.com" // OR a libp2p multiaddr string
}
```

**Behavior:**

* If `address` is an HTTP(S) address:

  * Fetches posts from `address + /_openherd/outbox`.
  * Imports them using local import logic.
  * Pushes up to 10,000 local posts via `POST` to `address + /_openherd/inbox`.
* If the HTTP request fails, attempts to dial `address` as a libp2p multiaddr.

**Response:**

```json
{ "ok": true, "message": "Sync complete" }
```

or in case of failure:

```json
{ "ok": false, "error": "Failed to dial multiaddr" }
```

## 7. Relay Nodes
Relay nodes are lightweight, non-interactive participants in the OpenHerd network. They do not run a libp2p node, and instead serve as passive endpoints that accept and serve posts via HTTP. Think of them as digital dead drops or temporary caches — great for use on open/public Wi-Fi, mesh networks, or constrained devices.

### Capabilities
Relays only implement the `/inbox` and `/outbox` endpoints (see Section 6). They do not participate in Gossipsub or direct libp2p message exchange.
This makes them ideal for:
- Public hotspots and darknet dropboxes
- Broadcast-only devices (e.g., Raspberry Pi on a local mesh)
- Interfacing with networks that don’t support libp2p

### Service Discovery (Bonjour)
Relays MAY advertise themselves on the local network using [Bonjour/mDNS](https://en.wikipedia.org/wiki/Bonjour_\(software\)):
- **Service Type**: `_http._tcp`
- **Name**: `openherd-relay`

Clients can scan for local relays and POST new messages to them automatically.

### Behavior
- Relays are expected to store received posts in `/inbox`, make them available via `/outbox`, and optionally forward them to the larger network via external tools or later sync.
- Posts on a relay are not guaranteed to be permanent. They may expire, be flushed, or be deleted on reboot.

## 8. Beacon Network

The Beacon Network is a centralized discovery mechanism that allows relay nodes to broadcast their presence to a public directory. This enables clients and other relays to locate OpenHerd endpoints operating in specific geographic areas or broadcasting specific SSIDs.

### Relay Registration

Relays MAY choose to register with the beacon server. Registration is cryptographically signed and proof-of-work gated to prevent abuse.

### Endpoint

```http
POST /beacon/update
```

**Request Body:**

```json
{
  "publicKey": "-----BEGIN PGP PUBLIC KEY BLOCK-----\n...",
  "message": "{\"lat\":\"33.5\",\"lng\":\"-84.2\",\"nickname\":\"peachtree-drop\",\"operator\":\"School of life\",\"ssid\":\"OpenHerd\",\"macAddress\":\"aa:bb:cc:dd:ee:ff\"}",
  "signature": "base64-encoded signature of the message",
  "nonce": 2379281
}
```

**Fields in `message`:**

- `lat`: Latitude (string)
- `lng`: Longitude (string)
- `nickname`: Human-friendly node name (e.g., `"peachtree-drop"`)
- `operator`: Who is running the relay (optional, pseudonyms allowed)
- `ssid`: SSID being broadcasted or connected to
- `macAddress`: MAC address of the relay’s network interface

### Authentication

Each registration must be:

- Signed using the relay’s long-term key (stored locally in `.data/.secretkey`)
- Validated with a Proof of Work (PoW) challenge:
    - SHA256(publicKey + nonce) must start with five leading zeros (`00000...`)
        

### Key Management

Relays are expected to persist their PGP keypair between sessions:

- Public key: `./.data/.publickey`
- Secret key: `./.data/.secretkey`

Use a CLI tool such as `keygen` in the relay software to generate keys.

### Example Registration Flow

```js
await register({
  lat: location.lat.toString(),
  lng: location.lng.toString(),
  nickname: config.nickname,
  operator: config.operator,
  ssid: wifi.ssid,
  macAddress: wifi.macAddress
})
```

- This is typically run on startup and every few minutes to keep the beacon listing fresh.
### Discovery

Users may browse relays via a web interface provided by the beacon server). This includes a map of active relay nodes with their SSID, nickname, and general location.
