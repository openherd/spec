## 1. Overview

OpenHerd is a decentralized, anonymous communication platform based on a FOSS P2P architecture. This document outlines the core technical specifications for post creation, message signing, and data transmission.

## 2. Basic Envelope

Each post or reply MUST generate a new PGP key, which serves as its unique identifier.

### Envelope Structure

```json
{
  "version": "1",
  "signature": "-----BEGIN PGP SIGNATURE-----...",
  "publicKey": "-----BEGIN PGP PUBLIC KEY BLOCK-----...",
  "id": "keyID",
  "data": "{\"latitude\":\"\",\"longitude\":\"\"}"
}
```

## 3. Creating a New Post

A new PGP key SHALL be created for each post and used to sign all messages associated with that post. Location data SHOULD be skewed by at least 1 km in a randomly generated direction to enhance anonymity.

### Transmission

New posts are sent via the `posts` Gossipsub topic.

### Parent-Child Structure

Each post references its parent by using the PGP key ID. For example:

- `null` (root post) → `8558e99c353bbac709e470b6342241c315fe352a` (reply) → `6a63e1d077d496691a6047a35ed0d5d1a959fb40` (nested reply)
    

### Post Format
```json
{
  "latitude": "33.7501",
  "longitude": "-84.3885",
  "text": "Hello world!",
  "date": "2025-01-30T07:26:06.506Z",
  "parent": "8558e99c353bbac709e470b6342241c315fe352a"
}
```

## 4. Catchup
Clients can request past posts by sending an empty JSON object `{}` to the `catchup` endpoint. Optionally, a `max` parameter can be included to specify the maximum number of posts to retrieve.
### Request Format

```json
{
"max": 200
}
```
### Response
Clients should subscribe to the `backlog` topic, where an array of envelopes containing past posts will be delivered.

## 5. Chunking

OpenHerd employs chunking to ensure reliable message transmission, particularly for large messages. Each chunk MUST follow this format:

### Chunk Format

```json
{
  "index": 0,
  "total": 6,
  "content": "whatever"
}
```

### Guidelines

- The `index` field is 0-based.
- The `total` field represents the total number of chunks.
- Each `content` field should contain up to 500 characters.
- Chunks must be reassembled in order upon receipt.
