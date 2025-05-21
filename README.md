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
