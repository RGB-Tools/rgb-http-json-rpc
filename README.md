# RGB HTTP JSON-RPC

In order to operate an RGB transfer it is necessary to exchange special files,
called consignments, that represent the offline data used to move asset rights.
Moreover, with NFTs and collectibles there could also be the need to exchange
media files.

While this can be achieved using the [Storm protocol], which is a powerful
protocol for distributed storage, it is preferable not to rely on a single file
exchange medium, in order for the RGB ecosystem to be as resilient as possible.

To provide a simple alternative, we defined this protocol. It can be easily
implemented in almost any programming language and can use any storage system.

The protocol uses a [JSON-RPC 2.0] interface over HTTP(s).
In order to be compliant with the protocol, an implementation must support all
the [protocol methods](#protocol-methods).

The protocol is supported as a consignment endpoint type in the `Invoice`
provided by [rgb-wallet](https://github.com/RGB-WG/rgb-wallet).

Protocol version is `1.0`.

The rationale behind the protocol design, its limitations and the changes with
respect to version `0.2` are described in the [v1.0 changes specification].


## Overview

A transfer involves two parties, each using a proxy server of its choice:
- the recipient, which reserves an invoice on its proxy (the recipient proxy),
  where the sender uploads the consignment and media files
- the sender, which registers the signed TX on its proxy (the sender proxy),
  where the recipient posts its signed acknowledgement (ACK) and, in exchange,
  obtains the signed TX

The recipient proxy URLs are included in the invoice as transport endpoints,
while the sender proxy URL is communicated to the recipient when the
consignment is uploaded.

The signed TX is always encrypted end-to-end between the two parties, while
consignment and media are encrypted according to the recipient preference and
may therefore be in plaintext (see [Encryption](#encryption) and
[Invoice](#invoice)). In both cases the proxies store and return opaque bytes,
without inspecting them.

An invoice can receive up to a quota of transfers (1 for a single-use invoice),
so the same invoice (and thus the same address or blinded UTXO) can be used for
multiple transfers.

### Transfer flow

```
Recipient                          Sender
    |                                |
    |-- choose keys                  |
    |                                |
    |-- reserve invoice on all       |
    |   recipient proxies            |
    |                                |
    |-- create invoice including --->|
    |   selected pubkey              |
    |                                |
    |                  choose keys --|
    |                                |
    |     check invoice can accept --|
    |   transfers on recipient proxy |
    |                                |
    |        compute proof of work --|
    |                                |
    |           post transfer data --|
    |            (inc. signed TXs)   |
    |              to sender proxy   |
    |                                |
    |     post signed consignment, --|
    |    related data and proof of   |
    |      work to recipient proxy   |
    |                                |
    |-- list invoice transfers on    |
    |   recipient proxy              |
    |                                |
    |-- get consignment and related  |
    |   data from recipient proxy    |
    |                                |
    |-- post signed ACK to sender    |
    |   proxy (and get signed TX)    |
    |                                |
    |-- broadcast TX                 |
    |                                |
    |  get signed ACKs from sender --|
    |         proxy and store them   |
    |                                |
    |    broadcast TX (idempotent) --|
    |                                |
    |--   wait for confirmations   --|
```

Donations are the exception: the signed TX is included in the consignment and
broadcast by the sender right after uploading it, so neither the sender proxy
nor the ACK are involved.


## Concepts

### Identifiers

- invoice: the recipient's request to receive (see [Invoice](#invoice)),
  reserved on the recipient proxy via [`invoice.prepare`](#invoiceprepare)
- invoice pubkey: the recipient pubkey included in the invoice; a fresh one is
  used for each invoice and it is the recipient pubkey for all the transfers to
  the invoice
- quota: the maximum number of transfers an invoice can receive
- transfer: a single transfer to an invoice, created on the recipient proxy by
  a successful [`consignment.post`](#consignmentpost) call
- beneficiary string: the canonical string form of the invoice beneficiary,
  i.e. the chain network prefix (e.g. `bc`, `tb4`, `bcrt`), a `:`, then either
  `utxob:` followed by the Baid64-encoded concealed seal (blind receives) or
  `wvout:` followed by the Baid64-encoded pay-to-vout payload (witness
  receives), in both cases including the `-` chunk separators, followed, for a
  witness beneficiary that includes an internal key, by `+` and the
  hex-encoded internal pubkey; the parsed beneficiary must be re-encoded in
  this form regardless of how it appears in the received invoice (e.g. without
  separators), and the plain bitcoin address must never be used
- invoice ID: hex-encoded SHA-256 of the concatenation of the literal string
  `rgb-proxy-server/invoice/v1`, the beneficiary string and the invoice pubkey
- transfer ID: hex-encoded SHA-256 of the concatenation of the literal string
  `rgb-proxy-server/transfer/v1`, the invoice ID and the sender pubkey
- operation: the set of transfers sent with a single bitcoin TX, registered on
  the sender proxy via [`ack.prepare`](#ackprepare)

Concatenations are made of the fields encoded as UTF-8 bytes, with no
separators.

Example IDs, with invoice pubkey
`5c298bf4b0dcf967871778ca682b4de838eb7a201853845c49adc15b10681360` and sender
pubkey `e11a179a775712d6763f01de1e10a6141c08f098816aa89a0da333974bf8bc2b`:

| receive | beneficiary string | invoice ID | transfer ID |
|---------|--------------------|------------|-------------|
| blind | `bc:utxob:4vm1CX2Z-K8hMo59-e7dgGBS-Jka7mYn-Xe~yP85-yUiHHxr-aVlYa` | `6ffae5a533bad53bb7ff50152f8a282564fb87429a53384f1b20e88895e7faad` | `285530722532e2fc802c0ad10bd381d3cca23f01af7d89fb03ec55f734a2a3e7` |
| witness | `bc:wvout:A8cJ7Ww3-NIzADo3-Tzp_5aD-7CTBWmA-AAAAAAA-AAAAAAA-ALSQkcw` | `4b4ef334f725afa5b09959a62518a09da375b0708c14f760745c241339b8d54f` | `7ac01c41928a6a114e3320aa4bcbdedb9dd7dd6915fcf993ecd5544fdce1c00f` |

### Encoding

- IDs, hashes and pubkeys are lowercase hex strings of 64 characters
- signatures are lowercase hex strings of 128 characters
- numbers (timestamps, sizes, quotas, difficulties) are carried as decimal
  strings, except the `uptime` of [`server.info`](#serverinfo), a JSON number:
  `server.info` keeps the shape of its pre-v1.0 fields so that any client can
  parse it and detect the protocol version
- timestamps are unix absolute timestamps, in seconds, UTC
- the signed TX and the returned files are standard base64 strings (with
  padding)
- hashes of files are SHA-256 hashes of their plaintext content

Methods uploading a file ([`consignment.post`](#consignmentpost) and
[`media.post`](#mediapost)) use a multipart form (see
[Protocol methods](#protocol-methods)).

### Keys and signatures

Each party holds an Ed25519 key pair, made of a secret key and the
corresponding public key (pubkey):
- the recipient generates a fresh key pair for each invoice
- the sender generates a fresh key pair for each operation

Secret keys are never transmitted, so every `pk` field (and the `pk` invoice
query parameter) always refers to a pubkey.

A proxy can optionally be configured with a list of authentication pubkeys,
belonging to its operator(s) or authorized users, to restrict calls to
[`invoice.prepare`](#invoiceprepare) and [`ack.prepare`](#ackprepare).

Every signature follows the same scheme:
- the message is the concatenation, encoded as UTF-8 bytes, of the fields
  defined for the method, with no separators
- the signed value is the 32-byte SHA-256 digest of the message
- the digest is signed with Ed25519 using the signer's secret key

| method | signer | message |
|--------|--------|---------|
| `invoice.prepare` (optional) | authentication key | `invoice_id`, `expires_at`, `recipient_pk`, quota (`1` if omitted) |
| `consignment.post` | sender key | `transfer_id`, `consignment_hash`, `blob_hash` |
| `media.post` | sender key | `transfer_id`, `media_id`, `blob_hash` |
| `transfer.list` | invoice key | `invoice_id`, `timestamp` |
| `transfer.retrieved` | invoice key | `transfer_id`, literal `retrieved` |
| `transfer.delete` | invoice key | `transfer_id`, literal `delete` |
| `ack.prepare` (optional) | authentication key | `sender_pk`, `expires_at`, `signed_tx_hash`, then for each transfer, sorted lexicographically by transfer ID: `transfer_id`, `recipient_pk`, `consignment_hash` |
| `ack.post` | invoice key | `transfer_id`, literal `true` or `false` (the ACK value), `consignment_hash` |
| `ack.get` | sender key | all the lexicographically sorted `transfer_ids`, `timestamp` |
| `ack.clean` | sender key | all the lexicographically sorted `transfer_ids` |

`blob_hash` is the hex-encoded SHA-256 of the uploaded (possibly encrypted)
file, which the proxy computes itself and is never transmitted.

The `timestamp` of [`transfer.list`](#transferlist) and [`ack.get`](#ackget)
must be within 300 seconds of the proxy clock (see `current_time` in
[`server.info`](#serverinfo)), otherwise the call is refused as invalid
([`-200`](#invalid--200), with `timestamp` among the fields), unlike an
out-of-window `pow_timestamp`, which gets
[`-114`](#invalid-proof-of-work--114).

### Expiration and retention

Invoices (via [`invoice.prepare`](#invoiceprepare)) and operations (via
[`ack.prepare`](#ackprepare)) are reserved with an expiration (`expires_at`),
which can't be further in the future than the proxy `max_expiry_secs` (see
[`server.info`](#serverinfo)).

After the expiration:
- no new transfers are accepted for an expired invoice and no signed TX is
  released for an expired operation
- the recipient proxy keeps the data of pending transfers (neither retrieved
  nor deleted by the recipient) for `retention_grace_secs`, along with the
  invoice reservation and its transfer records, needed to authenticate the
  recipient calls; once no pending transfer is left, all the invoice data can
  be deleted
- the sender proxy deletes the signed TXs of operations not ACKed by all
  recipients, while it keeps the operation data (ACK values, signatures and
  released signed TXs) for `retention_grace_secs`
- after the retention grace, all the data can be deleted
- an invoice reservation is never deleted before its expiration


## Encryption

Consignment, media and signed TX are encrypted end-to-end between sender and
recipient. Consignment and media encryption follows the recipient preference
(see [Invoice](#invoice)), while the signed TX is always encrypted.

### Key derivation

Both parties derive the same shared secret via X25519 ECDH between their own
secret key and the counterpart pubkey, converted from Ed25519 to X25519 with the
standard Curve25519 birational map (RFC 7748):
- pubkey: the X25519 pubkey is the Montgomery `u` coordinate,
  `u = (1 + y) / (1 - y) mod p`, with `p = 2^255 - 19`, encoded as a 32-byte
  little-endian value
- secret key: the Ed25519 secret key is the 32-byte seed (the RFC 8032 private
  key, not libsodium's 64-byte secret key); the X25519 secret scalar is
  `clamp(SHA-512(seed)[0..32])`, with standard Curve25519 clamping

Implementations should use a vetted conversion routine (e.g. libsodium's
`crypto_sign_ed25519_pk_to_curve25519` / `crypto_sign_ed25519_sk_to_curve25519`,
`@noble/curves` `edwardsToMontgomery*` or `curve25519-dalek` `to_montgomery`).

The symmetric key is derived with HKDF-SHA-256 (RFC 5869):

```
PRK = HKDF-Extract(salt = <empty>, IKM = shared_secret)
key = HKDF-Expand(PRK, info = "rgb-proxy-server/enc/v1", L = 32)
```

The same key is used for consignment, media and signed TX.

### Encrypted blob format

Data is encrypted with XChaCha20-Poly1305 (one-shot AEAD, caller-supplied
nonce), in chunks of 65536 plaintext bytes. Each chunk is encrypted
independently and followed by its 16-byte tag. A plaintext of `n` bytes is
split into `max(1, ceil(n / 65536))` chunks:
- every chunk except the last is exactly 65536 bytes
- the last (final) chunk holds the remaining bytes, from 1 to 65536 (a full
  65536-byte final chunk when `n` is a multiple of 65536, with no additional
  empty chunk)
- an empty plaintext is encrypted as a single empty final chunk (just its
  16-byte tag)

Each 24-byte chunk nonce is the concatenation of:
- a 19-byte base: the last 19 bytes of the raw (hex-decoded) SHA-256 hash of
  the plaintext (`consignment_hash`, `signed_tx_hash` or the media digest)
- a 4-byte big-endian chunk counter, starting at 0
- a 1-byte final-chunk flag: `01` for the last chunk, `00` otherwise

The blob is prefixed with a single-byte encryption-format version, currently
`0x01`, which selects the HKDF `info` label (`rgb-proxy-server/enc/v1`).

Example, for a 204800-byte plaintext:

| chunk | plaintext range (bytes) | plaintext size | counter | final flag | ciphertext size (incl. tag) |
|-------|-------------------------|----------------|---------|------------|-----------------------------|
| 0 |      0 -  65536 | 65536 | `00 00 00 00` | `00` | 65552 |
| 1 |  65536 - 131072 | 65536 | `00 00 00 01` | `00` | 65552 |
| 2 | 131072 - 196608 | 65536 | `00 00 00 02` | `00` | 65552 |
| 3 | 196608 - 204800 |  8192 | `00 00 00 03` | `01` |  8208 |

```
[0x01]             1 byte    version prefix
[chunk 0 || tag]   65552 bytes
[chunk 1 || tag]   65552 bytes
[chunk 2 || tag]   65552 bytes
[chunk 3 || tag]    8208 bytes   (final chunk, nonce final flag = 0x01)
```

To decrypt, the recipient reads the version prefix, derives the same key and
nonce base, then decrypts the 65552-byte ciphertext chunks in order, verifying
each tag. The final chunk, decrypted with the final flag `01`, is the one that
ends the blob, identified by its position rather than by its size, since it can
also be a full 65552-byte chunk.

### Media ID

Media attachments are identified by a `media_id`, derived from the shared
secret rather than being the plaintext digest:

```
Kmid     = HKDF-Expand(PRK, info = "rgb-proxy-server/enc/v1/media-id", L = 32)
media_id = hex(HMAC-SHA256(Kmid, media_digest))
```

where `media_digest` is the hex-encoded SHA-256 of the plaintext media, known
to both parties from the asset metadata and never sent to the proxy.
`media_id` is used for all media, both encrypted and cleartext.

### Witness seal blinding

For witness receives, the blinding factor of the seal assigning assets to the
recipient output is derived from the shared secret instead of being chosen by
the sender:

```
Kbld     = HKDF-Expand(PRK, info = "rgb-proxy-server/enc/v1/seal-blinding", L = 8)
blinding = u64_le(Kbld)
```

where `u64_le` interprets the 8 bytes as an unsigned 64-bit little-endian
integer.

The sender uses `blinding` for the recipient witness seal, while the recipient
uses it, together with the script and the vout of the output, to identify the
output paying the invoice. For blind receives, the blinding factor is chosen by
the recipient and embedded in the invoice concealed seal.


## Invoice

The invoice (defined by the RGB invoice standard) carries, as query
parameters:
- `pk`: the invoice pubkey (mandatory)
- `enc` (optional): the recipient consignment and media encryption preference:
    - missing: encryption is mandatory, cleartext files are refused
    - `0`: encryption must not be used, encrypted files are refused
    - `1`: encryption may be used, both are accepted (senders should default to
      encryption)

The invoice expiration is mandatory.

When creating an invoice, the recipient:
- generates a fresh key pair for the invoice and stores it (the seed)
- sets the invoice expiration, which must be the same as the `expires_at` used
  to reserve the invoice on the recipient proxies, hence not further in the
  future than the lowest `max_expiry_secs` among them (minus some margin for
  clock skew)
- reserves the invoice on all the transport endpoints it lists, via
  [`invoice.prepare`](#invoiceprepare), with the same expiration, invoice pubkey
  and quota, before sharing the invoice; if a proxy refuses the reservation, it
  either lowers the expiration so that all proxies accept it or removes that
  transport endpoint from the invoice

When parsing an invoice, the sender:
- considers it invalid if `pk` or the expiration is missing
- encrypts the consignment and media according to `enc`, or refuses the invoice
  if the requested mode is not acceptable
- computes the invoice ID and calls [`invoice.check`](#invoicecheck) to learn
  whether the invoice can accept a transfer, since nothing in the invoice
  signals its quota
- uses the pubkey and expiration from the invoice, never the data registered
  on the recipient proxy

Each proxy enforces the quota independently, so an invoice listing more than
one transport endpoint can receive up to the sum of their quotas.


## Proof of work

The recipient proxy can require a proof of work on
[`consignment.post`](#consignmentpost), with a difficulty `d` that is 0 during
normal operation and rises while an invoice or the proxy is under attack.
The current difficulty for an invoice is returned by
[`invoice.check`](#invoicecheck) and the maximum one is advertised by
[`server.info`](#serverinfo) (`0` means proof of work is disabled).

The sender must find a `pow_nonce` (1 to 64 printable ASCII characters) such
that:

```
SHA-256(transfer_id || pow_timestamp || pow_nonce)
```

has at least `d` leading zero bits, where the input is the concatenation of the
three fields encoded as UTF-8 bytes, with no separators, and leading zero bits
are counted from the most significant bit of the first byte of the digest.

`pow_timestamp` must be within 300 seconds of the proxy clock.

`pow_timestamp` and `pow_nonce` are always required: when the difficulty is 0
any value is accepted. If the proof of work does not satisfy the current
difficulty, or `pow_timestamp` is outside the allowed window, the proxy returns
an [invalid proof of work](#invalid-proof-of-work--114) error including the
current difficulty, and the sender computes a new proof of work (with a fresh
`pow_timestamp`) and retries, with no need to rebuild the operation.

The recommended `max_pow_difficulty` is 24, which should take roughly 1 second
with native code on a mid-range phone and 3 to 8 seconds in a mobile browser.
Clients can refuse to compute proofs of work above their own limits.


## Protocol methods

| method | caller | proxy |
|--------|--------|-------|
| [`server.info`](#serverinfo) | both | both |
| [`invoice.prepare`](#invoiceprepare) | recipient | recipient |
| [`invoice.check`](#invoicecheck) | sender | recipient |
| [`consignment.post`](#consignmentpost) | sender | recipient |
| [`consignment.get`](#consignmentget) | recipient | recipient |
| [`media.post`](#mediapost) | sender | recipient |
| [`media.get`](#mediaget) | recipient | recipient |
| [`transfer.list`](#transferlist) | recipient | recipient |
| [`transfer.retrieved`](#transferretrieved) | recipient | recipient |
| [`transfer.delete`](#transferdelete) | recipient | recipient |
| [`ack.prepare`](#ackprepare) | sender | sender |
| [`ack.post`](#ackpost) | recipient | sender |
| [`ack.get`](#ackget) | sender | sender |
| [`ack.clean`](#ackclean) | sender | sender |

All methods are JSON-RPC 2.0 requests sent via HTTP `POST` to the proxy
endpoint (e.g. `https://proxy.example.com/json-rpc`), with an
`application/json` body, except the methods uploading a file
([`consignment.post`](#consignmentpost) and [`media.post`](#mediapost)), which
use a `multipart/form-data` body: the file is attached under the `file` name,
while the JSON-RPC fields are sent as form fields, with `params` as a
JSON-encoded string, which must precede the file.

Upload checks, applied while receiving the request and right after the
parameter validation:
- a file part arriving before `params` is refused ([`-200`](#invalid--200),
  with `file` among the fields)
- a declared `file_size` above `max_file_size` (see
  [`server.info`](#serverinfo)) is refused ([`-108`](#file-too-large--108))
- a missing file is refused ([`-301`](#missing-file--301))
- the received file must be exactly `file_size` bytes, otherwise it's refused
  ([`-200`](#invalid--200), with `file_size` among the fields); a larger file
  is aborted as soon as it exceeds the declared size

When more than one validation error is possible, missing parameters are
reported first ([`-300`](#missing--300)) and invalid ones
([`-200`](#invalid--200)) only once all parameters are present. Both errors
list all the failing fields.

### server.info

#### request

HTTP `POST`, `application/json` body.

```json
{ "id": 0, "method": "server.info" }
```

#### response

```json
{
  "id": 0,
  "result": {
    "protocol_version": "1.0",
    "version": "1.0.0",
    "uptime": 42069,
    "current_time": "1791367200",
    "max_expiry_secs": "2592000",
    "max_quota": "100",
    "max_pow_difficulty": "24",
    "retention_grace_secs": "2592000",
    "max_file_size": "104857600",
    "max_signed_tx_size": "153600",
    "max_transfers": "1000"
  }
}
```

Result:
- `protocol_version`: the supported protocol version
- `version`: the server version
- `uptime`: the server uptime, in seconds
- `current_time`: the proxy clock, which clients can use to compute the clock
  skew
- `max_expiry_secs`: the maximum number of seconds in the future an
  `expires_at` can be set to
- `max_quota`: the maximum quota an invoice can be reserved with
- `max_pow_difficulty`: the maximum proof of work difficulty the proxy can
  require (`0` if proof of work is disabled)
- `retention_grace_secs`: how long, after the expiration, the proxy
  guarantees to keep data awaiting retrieval (`0` if no grace is granted)
- `max_file_size`: the maximum size, in bytes, of a file uploaded via
  [`consignment.post`](#consignmentpost) or [`media.post`](#mediapost)
- `max_signed_tx_size`: the maximum size of each signed TX in
  [`ack.prepare`](#ackprepare), measured as the length of its base64 string
- `max_transfers`: the maximum number of transfers an
  [`ack.prepare`](#ackprepare) call can include

### invoice.prepare

Reserve an invoice, before sharing it, so that it can receive transfers.

#### request

HTTP `POST`, `application/json` body.

```json
{
  "id": 1,
  "method": "invoice.prepare",
  "params": {
    "invoice_id": "a39abbd52b82bace937079982fa585941ce084a045d92bdb443643bc98235b3f",
    "expires_at": "1791972000",
    "recipient_pk": "5c298bf4b0dcf967871778ca682b4de838eb7a201853845c49adc15b10681360",
    "quota": "1"
  }
}
```

Params:
- `invoice_id`: the invoice ID
- `expires_at`: the invoice expiration, which must be in the future and not
  exceed `max_expiry_secs`
- `recipient_pk`: the invoice pubkey, which authenticates the recipient on
  [`transfer.list`](#transferlist), [`transfer.retrieved`](#transferretrieved)
  and [`transfer.delete`](#transferdelete)
- `quota`: (optional) the maximum number of transfers (default `1`), between 1
  and `max_quota`
- `signature`: (optional) required only if the proxy is configured with
  authentication pubkeys (see [Keys and signatures](#keys-and-signatures))

#### response

When the invoice is newly reserved:
```json
{ "id": 1, "result": true }
```

When the invoice was already reserved with the same data:
```json
{ "id": 1, "result": false }
```

Behavior:
- an invoice ID can be reserved only once and the reservation can't be changed
  nor removed before its expiration
- the proxy cannot verify that `recipient_pk` matches the invoice ID: whoever
  reserves the invoice ID first owns the reservation

Errors:
- [`-100`](#cannot-change--100)
- [`-115`](#server-draining--115)
- [`-200`](#invalid--200)
- [`-300`](#missing--300)
- [`-501`](#invalid-proxy-auth-signature--501)

### invoice.check

Check whether one or more invoices can accept transfers and get the proof of
work difficulty currently required.

#### request

HTTP `POST`, `application/json` body.

```json
{
  "id": 2,
  "method": "invoice.check",
  "params": {
    "invoice_ids": [
      "a39abbd52b82bace937079982fa585941ce084a045d92bdb443643bc98235b3f"
    ]
  }
}
```

Params:
- `invoice_ids`: the invoice IDs to check

#### response

```json
{
  "id": 2,
  "result": {
    "a39abbd52b82bace937079982fa585941ce084a045d92bdb443643bc98235b3f": {
      "status": "available",
      "pow_difficulty": "0"
    }
  }
}
```

Result: a map from invoice ID to:
- `status`:
    - `unknown`: the invoice is not known to the proxy (not reserved, or its
      data has been deleted)
    - `available`: the invoice is reserved and its quota is not full
    - `full`: the invoice is reserved and its quota is full
- `pow_difficulty`: the proof of work difficulty currently required to create
  a transfer for the invoice

Behavior:
- the check is read-only: an `available` status doesn't guarantee that a later
  [`consignment.post`](#consignmentpost) will succeed
- the status doesn't consider the invoice expiration, which the sender knows
  from the invoice

Senders should check each invoice on all the transport endpoints it lists,
typically stopping at the first one that returns `available`.

Errors:
- [`-200`](#invalid--200)
- [`-300`](#missing--300)

### consignment.post

Upload a consignment, creating a transfer to an invoice.

#### request

HTTP `POST`, `multipart/form-data` body.

In addition to the parameters, the (possibly encrypted) consignment file needs
to be attached under the `file` name via multipart form.

```json
{
  "id": 3,
  "method": "consignment.post",
  "params": {
    "invoice_id": "a39abbd52b82bace937079982fa585941ce084a045d92bdb443643bc98235b3f",
    "transfer_id": "6313972fcf4dfede897ed8cd932c579af0eb83c57ff8a8f6def81412ccd4ae0e",
    "sender_transport_endpoint": "rpc://proxy.example.com/json-rpc",
    "sender_pk": "e11a179a775712d6763f01de1e10a6141c08f098816aa89a0da333974bf8bc2b",
    "consignment_hash": "0949f665e0563c0a769aee4dbea0f71f958b183985ab892e7745d3c67a0560a0",
    "batch": false,
    "file_size": "48213",
    "signature": "e7e8be3d72de57e64156e95f2631a30a9eaa0e1a98863c2f1512affa5056f7dc9ca86f117e5621a817e6c8aebf1121f584c68bd5e5666aab2e85662632ef8fb6",
    "pow_timestamp": "1791367210",
    "pow_nonce": "0"
  }
}
```

Params:
- `invoice_id`: the invoice ID of the transfer
- `transfer_id`: the transfer ID, derived from `invoice_id` and `sender_pk`
- `sender_transport_endpoint`: the sender proxy, in the `RgbTransport` format
  (e.g. `rpc://...`), where the recipient will post its ACK
- `sender_pk`: the sender pubkey of the operation
- `consignment_hash`: the hex-encoded SHA-256 of the plaintext consignment
- `batch`: whether the operation includes more than one transfer
- `file_size`: the size of the uploaded file, in bytes
- `signature`: the sender signature (see
  [Keys and signatures](#keys-and-signatures))
- `pow_timestamp`: the proof of work timestamp (see
  [Proof of work](#proof-of-work))
- `pow_nonce`: the proof of work nonce

For donations, the signed TX is included in the consignment.

#### response

When the transfer is newly created:
```json
{ "id": 3, "result": true }
```

When the identical transfer was already created:
```json
{ "id": 3, "result": false }
```

Behavior, in order:
1. missing, then invalid parameters, then the
   [upload checks](#protocol-methods)
2. an identical re-post returns `false`, even after
   [`transfer.retrieved`](#transferretrieved), without checking the proof of
   work
3. the invoice must be reserved ([`-101`](#invoice-not-reserved--101)) and not
   expired ([`-111`](#invoice-expired--111)), `transfer_id` must match the one
   derived from `invoice_id` and `sender_pk` ([`-200`](#invalid--200)), the
   signature must be valid ([`-502`](#invalid-endpoint-auth-signature--502))
   and the transfer ID must not have been deleted
   ([`-113`](#transfer-id-deleted--113))
4. if the transfer ID already exists with different data, the call is rejected
   ([`-100`](#cannot-change--100), listing the differing fields)
5. the invoice quota must not be full ([`-112`](#invoice-quota-exceeded--112))
6. only if the current difficulty is greater than 0, the proof of work must be
   valid ([`-114`](#invalid-proof-of-work--114))

Errors:
- [`-100`](#cannot-change--100)
- [`-101`](#invoice-not-reserved--101)
- [`-108`](#file-too-large--108)
- [`-111`](#invoice-expired--111)
- [`-112`](#invoice-quota-exceeded--112)
- [`-113`](#transfer-id-deleted--113)
- [`-114`](#invalid-proof-of-work--114)
- [`-115`](#server-draining--115)
- [`-200`](#invalid--200)
- [`-300`](#missing--300)
- [`-301`](#missing-file--301)
- [`-502`](#invalid-endpoint-auth-signature--502)

### consignment.get

Get a consignment and the related data.

#### request

HTTP `POST`, `application/json` body.

```json
{
  "id": 4,
  "method": "consignment.get",
  "params": {
    "transfer_id": "6313972fcf4dfede897ed8cd932c579af0eb83c57ff8a8f6def81412ccd4ae0e"
  }
}
```

Params:
- `transfer_id`: the transfer ID

#### response

When a consignment is available, it is returned in base64-encoded form:
```json
{
  "id": 4,
  "result": {
    "consignment": "AQ8AHQEE/////wEA8gUqAQAAAENBBJa1OOhTUZxyapEeYewRYACuE5CBOmJ8ZvuL55R75jxS2nWJN5UV1OCmBPgUF4HiKUchFmv2Ih5zqCy/I0LIWO6sAAAAAA==",
    "sender_transport_endpoint": "rpc://proxy.example.com/json-rpc",
    "sender_pk": "e11a179a775712d6763f01de1e10a6141c08f098816aa89a0da333974bf8bc2b",
    "consignment_hash": "0949f665e0563c0a769aee4dbea0f71f958b183985ab892e7745d3c67a0560a0",
    "batch": false
  }
}
```

Result:
- `consignment`: the (possibly encrypted) consignment file
- `sender_transport_endpoint`: the sender proxy, where to post the ACK
- `sender_pk`: the sender pubkey, needed to decrypt and to verify the transfer
  ID
- `consignment_hash`: the hex-encoded SHA-256 of the plaintext consignment
- `batch`: whether the operation includes more than one transfer

Errors:
- [`-200`](#invalid--200)
- [`-300`](#missing--300)
- [`-400`](#consignment-not-found--400)
- [`-403`](#transfer-data-retrieved--403)

### media.post

Upload a media attachment for a transfer.

#### request

HTTP `POST`, `multipart/form-data` body.

In addition to the parameters, the (possibly encrypted) media file needs to be
attached under the `file` name via multipart form.

```json
{
  "id": 5,
  "method": "media.post",
  "params": {
    "transfer_id": "6313972fcf4dfede897ed8cd932c579af0eb83c57ff8a8f6def81412ccd4ae0e",
    "media_id": "317da33ac74b113d3b315789a0ac5a7bcdc65d1b1b2e828a9b18aba185370756",
    "file_size": "102400",
    "signature": "3835b11be35eba3ac2f18624e2667be619d3b65170cb440a53b64ef15e2b864c48768de9fcd7c8fae7c90c0dc3d0a14b3f0f700316e842c31fd288a4a2fea68c"
  }
}
```

Params:
- `transfer_id`: the transfer ID, which must have been created via
  [`consignment.post`](#consignmentpost)
- `media_id`: the media ID (see [Media ID](#media-id))
- `file_size`: the size of the uploaded file, in bytes
- `signature`: the sender signature, made with the key of the `sender_pk` of
  the transfer (see [Keys and signatures](#keys-and-signatures))

#### response

On first successful upload:
```json
{ "id": 5, "result": true }
```

On successive successful uploads (of the same media file):
```json
{ "id": 5, "result": false }
```

Behavior:
- parameters are validated first, then the [upload checks](#protocol-methods)
  are applied
- media can be posted while the transfer exists, also after the invoice
  expiration (e.g. for a transfer created right before it), as long as its
  data has not been retrieved ([`-403`](#transfer-data-retrieved--403))
- an identical re-post returns `false`, also after
  [`transfer.retrieved`](#transferretrieved)

Errors:
- [`-100`](#cannot-change--100)
- [`-108`](#file-too-large--108)
- [`-200`](#invalid--200)
- [`-300`](#missing--300)
- [`-301`](#missing-file--301)
- [`-402`](#transfer-not-found--402)
- [`-403`](#transfer-data-retrieved--403)
- [`-502`](#invalid-endpoint-auth-signature--502)

### media.get

Get a media attachment.

#### request

HTTP `POST`, `application/json` body.

```json
{
  "id": 6,
  "method": "media.get",
  "params": {
    "transfer_id": "6313972fcf4dfede897ed8cd932c579af0eb83c57ff8a8f6def81412ccd4ae0e",
    "media_id": "317da33ac74b113d3b315789a0ac5a7bcdc65d1b1b2e828a9b18aba185370756"
  }
}
```

Params:
- `transfer_id`: the transfer ID
- `media_id`: the media ID

#### response

When a media is available, the (possibly encrypted) file is returned in
base64-encoded form:
```json
{
  "id": 6,
  "result": "AQsAHdEE+P9/AQDyBSoBAAAAQ0EElrU46FNRnHJqkR5h7BFgAK4TkIE6YnxmPx+NBVpdTyaovx4fsKR2AXCzvlRvXd9cVZpKAQ=="
}
```

Errors:
- [`-200`](#invalid--200)
- [`-300`](#missing--300)
- [`-401`](#media-not-found--401)
- [`-403`](#transfer-data-retrieved--403)

### transfer.list

List the transfers of an invoice.

#### request

HTTP `POST`, `application/json` body.

```json
{
  "id": 7,
  "method": "transfer.list",
  "params": {
    "invoice_id": "a39abbd52b82bace937079982fa585941ce084a045d92bdb443643bc98235b3f",
    "timestamp": "1791367500",
    "signature": "82763abbe89e96738d25c643c22e2cddc8381b6cddfa8ce61d59bfc3798eb15db642285928605af3cbcb8b887780d72f9124a6003ba006a0811782d55dc39ef7"
  }
}
```

Params:
- `invoice_id`: the invoice ID
- `timestamp`: the current time, within 300 seconds of the proxy clock
- `signature`: the recipient signature, made with the invoice key (see
  [Keys and signatures](#keys-and-signatures))

#### response

```json
{
  "id": 7,
  "result": {
    "6313972fcf4dfede897ed8cd932c579af0eb83c57ff8a8f6def81412ccd4ae0e": "pending",
    "ccde6c4ddc6ebc8add7a88f1f917d85f01881a1cd855a9cdaf87d6fa1b3b38c8": "retrieved"
  }
}
```

Result: a map from transfer ID to its state:
- `pending`: the transfer data (consignment and media) is available
- `retrieved`: the transfer data has been deleted via
  [`transfer.retrieved`](#transferretrieved), while the transfer still counts
  towards the invoice quota

Deleted transfers are not listed.

Errors:
- [`-101`](#invoice-not-reserved--101)
- [`-200`](#invalid--200)
- [`-300`](#missing--300)
- [`-502`](#invalid-endpoint-auth-signature--502)

The [`-101`](#invoice-not-reserved--101) error is returned before verifying the
signature, allowing the recipient to distinguish a reservation lost in a proxy
data wipe from one taken by someone else. After the invoice expiration, it can
also mean that the proxy deleted the invoice data because no transfer was
pending.

### transfer.retrieved

Signal that the data of a transfer has been retrieved and saved, so the proxy
deletes its consignment and media, while the transfer keeps counting towards
the invoice quota.

#### request

HTTP `POST`, `application/json` body.

```json
{
  "id": 8,
  "method": "transfer.retrieved",
  "params": {
    "transfer_id": "6313972fcf4dfede897ed8cd932c579af0eb83c57ff8a8f6def81412ccd4ae0e",
    "signature": "82763abbe89e96738d25c643c22e2cddc8381b6cddfa8ce61d59bfc3798eb15db642285928605af3cbcb8b887780d72f9124a6003ba006a0811782d55dc39ef7"
  }
}
```

Params:
- `transfer_id`: the transfer ID
- `signature`: the recipient signature, made with the invoice key (see
  [Keys and signatures](#keys-and-signatures))

#### response

When the transfer has been marked as retrieved:
```json
{ "id": 8, "result": true }
```

When the transfer had already been marked as retrieved or deleted:
```json
{ "id": 8, "result": false }
```

Behavior:
- after this call [`consignment.get`](#consignmentget) and
  [`media.get`](#mediaget) return [`-403`](#transfer-data-retrieved--403), while
  identical re-posts are still recognized
- a retrieved transfer can still be deleted via
  [`transfer.delete`](#transferdelete)

Errors:
- [`-200`](#invalid--200)
- [`-300`](#missing--300)
- [`-402`](#transfer-not-found--402)
- [`-502`](#invalid-endpoint-auth-signature--502)

The [`-402`](#transfer-not-found--402) error is returned before verifying the
signature, since the pubkey to verify it with is only known for an existing
transfer.

### transfer.delete

Delete a transfer (e.g. a settled, refused or fake one), removing its
consignment and media and freeing one slot of the invoice quota.

#### request

HTTP `POST`, `application/json` body.

```json
{
  "id": 9,
  "method": "transfer.delete",
  "params": {
    "transfer_id": "6313972fcf4dfede897ed8cd932c579af0eb83c57ff8a8f6def81412ccd4ae0e",
    "signature": "82763abbe89e96738d25c643c22e2cddc8381b6cddfa8ce61d59bfc3798eb15db642285928605af3cbcb8b887780d72f9124a6003ba006a0811782d55dc39ef7"
  }
}
```

Params:
- `transfer_id`: the transfer ID
- `signature`: the recipient signature, made with the invoice key (see
  [Keys and signatures](#keys-and-signatures))

#### response

When the transfer has been deleted:
```json
{ "id": 9, "result": true }
```

When the transfer had already been deleted:
```json
{ "id": 9, "result": false }
```

Behavior:
- a transfer can be deleted both while pending and after being retrieved
- a deleted transfer ID can't be used again (see
  [`-113`](#transfer-id-deleted--113))
- the invoice reservation is never deleted by this call

Errors:
- [`-200`](#invalid--200)
- [`-300`](#missing--300)
- [`-402`](#transfer-not-found--402)
- [`-502`](#invalid-endpoint-auth-signature--502)

The [`-402`](#transfer-not-found--402) error is returned before verifying the
signature, since the pubkey to verify it with is only known for an existing
transfer.

### ack.prepare

Register an operation on the sender proxy, with the data needed to verify the
recipients' ACKs and the signed TX to release once all of them have ACKed.
Not needed for donations.

#### request

HTTP `POST`, `application/json` body.

```json
{
  "id": 10,
  "method": "ack.prepare",
  "params": {
    "transfer_data": {
      "6313972fcf4dfede897ed8cd932c579af0eb83c57ff8a8f6def81412ccd4ae0e": {
        "recipient_pk": "5c298bf4b0dcf967871778ca682b4de838eb7a201853845c49adc15b10681360",
        "consignment_hash": "0949f665e0563c0a769aee4dbea0f71f958b183985ab892e7745d3c67a0560a0",
        "signed_tx": "AQHrCMd+mQ6ZeYBnUo2Jf7d8pY1tL4bX2hKzQ0vV3wQ5E6iRgS1u8aPcN7yTjW+0fA=="
      }
    },
    "expires_at": "1791710000",
    "signed_tx_hash": "cbcc7a7e01884c0493372da517f21cf52d90bf9715868086a1bdbaea5ddde5f6",
    "sender_pk": "e11a179a775712d6763f01de1e10a6141c08f098816aa89a0da333974bf8bc2b"
  }
}
```

Params:
- `transfer_data`: a map from transfer ID to:
    - `recipient_pk`: the invoice pubkey of the transfer
    - `consignment_hash`: the hex-encoded SHA-256 of the plaintext consignment
    - `signed_tx`: the signed TX, encrypted for the recipient, base64-encoded
- `expires_at`: the operation expiration, which must be in the future and not
  exceed `max_expiry_secs`
- `signed_tx_hash`: the hex-encoded SHA-256 of the plaintext signed TX (the
  same for all transfers)
- `sender_pk`: the sender pubkey of the operation, which authenticates the
  sender on [`ack.get`](#ackget) and [`ack.clean`](#ackclean)
- `signature`: (optional) required only if the proxy is configured with
  authentication pubkeys (see [Keys and signatures](#keys-and-signatures))

#### response

When the operation is newly registered:
```json
{ "id": 10, "result": true }
```

When the identical operation was already registered:
```json
{ "id": 10, "result": false }
```

Behavior:
- a transfer ID can belong to a single operation: a request including a
  transfer ID already registered to a different operation, or only a subset of
  the transfer IDs of an operation, is refused
  ([`-102`](#transfer-id-in-use--102))
- a request with the same transfer IDs as an existing operation but a
  differing field is refused ([`-100`](#cannot-change--100), listing the
  differing fields)
- the proxy refuses requests with more than `max_transfers` transfers or with a
  signed TX larger than `max_signed_tx_size` (see [`server.info`](#serverinfo))
- an identical re-post returns `false` only until the expiration and while the
  signed TXs have not been removed (by [`ack.clean`](#ackclean) or after a
  NACK); after that it's refused ([`-200`](#invalid--200) or
  [`-100`](#cannot-change--100))

Errors:
- [`-100`](#cannot-change--100)
- [`-102`](#transfer-id-in-use--102)
- [`-109`](#too-many-transfers--109)
- [`-110`](#signed-tx-too-large--110)
- [`-115`](#server-draining--115)
- [`-200`](#invalid--200)
- [`-300`](#missing--300)
- [`-501`](#invalid-proxy-auth-signature--501)

### ack.post

Post the signed ACK (accept) or NACK (reject) for a transfer and, on ACK, get
the signed TX.

#### request

HTTP `POST`, `application/json` body.

To accept a transfer:

```json
{
  "id": 11,
  "method": "ack.post",
  "params": {
    "transfer_id": "6313972fcf4dfede897ed8cd932c579af0eb83c57ff8a8f6def81412ccd4ae0e",
    "ack": true,
    "signature": "e7e8be3d72de57e64156e95f2631a30a9eaa0e1a98863c2f1512affa5056f7dc9ca86f117e5621a817e6c8aebf1121f584c68bd5e5666aab2e85662632ef8fb6"
  }
}
```

To reject a transfer:

```json
{
  "id": 11,
  "method": "ack.post",
  "params": {
    "transfer_id": "6313972fcf4dfede897ed8cd932c579af0eb83c57ff8a8f6def81412ccd4ae0e",
    "ack": false,
    "signature": "3835b11be35eba3ac2f18624e2667be619d3b65170cb440a53b64ef15e2b864c48768de9fcd7c8fae7c90c0dc3d0a14b3f0f700316e842c31fd288a4a2fea68c"
  }
}
```

Params:
- `transfer_id`: the transfer ID
- `ack`: whether the transfer is accepted (`true`) or rejected (`false`)
- `signature`: the recipient signature, made with the invoice key (see
  [Keys and signatures](#keys-and-signatures))

#### response

On ACK, when the signed TX is released:
```json
{
  "id": 11,
  "result": {
    "result": true,
    "sender_pk": "e11a179a775712d6763f01de1e10a6141c08f098816aa89a0da333974bf8bc2b",
    "signed_tx_hash": "cbcc7a7e01884c0493372da517f21cf52d90bf9715868086a1bdbaea5ddde5f6",
    "signed_tx": "AQHrCMd+mQ6ZeYBnUo2Jf7d8pY1tL4bX2hKzQ0vV3wQ5E6iRgS1u8aPcN7yTjW+0fA==",
    "expires_at": "1791710000"
  }
}
```

On ACK, when the signed TX is not (yet) released:
```json
{
  "id": 11,
  "result": {
    "result": true,
    "sender_pk": "e11a179a775712d6763f01de1e10a6141c08f098816aa89a0da333974bf8bc2b",
    "signed_tx_hash": "cbcc7a7e01884c0493372da517f21cf52d90bf9715868086a1bdbaea5ddde5f6",
    "expires_at": "1791710000"
  }
}
```

On NACK:
```json
{
  "id": 11,
  "result": {
    "result": true,
    "expires_at": "1791710000"
  }
}
```

Result:
- `result`: `true` on the first successful post, `false` on successive posts
  of the same ACK value and signature
- `sender_pk`: (only on ACK) the sender pubkey registered via
  [`ack.prepare`](#ackprepare), to be used to decrypt the signed TX
- `signed_tx_hash`: (only on ACK) the hex-encoded SHA-256 of the plaintext
  signed TX, needed to decrypt and verify the signed TX
- `signed_tx`: (optional) the signed TX, encrypted for the recipient,
  base64-encoded
- `expires_at`: the operation expiration, which, together with the sender
  proxy's `retention_grace_secs`, sets the broadcast deadline (see
  [Recipient](#recipient))

Behavior:
- the proxy verifies the signature against the `recipient_pk` and the
  `consignment_hash` registered for the transfer
- the signed TX is released only once all the transfers of the operation have
  been ACKed (the last recipient to ACK gets it right away, the others by
  posting the same ACK again) and only before the operation expiration
- after a NACK, the signed TX is never released
- a new ACK or NACK (not an identical re-post) for an expired operation is
  refused ([`-105`](#operation-expired--105))
- successive posts with the same data return the same data, also after the
  expiration and until the end of the retention grace, including a released
  signed TX (an ACK that would still be waiting for the signed TX instead
  returns [`-105`](#operation-expired--105) once the operation has expired)

Errors:
- [`-100`](#cannot-change--100)
- [`-104`](#operation-not-found--104)
- [`-105`](#operation-expired--105)
- [`-200`](#invalid--200)
- [`-300`](#missing--300)
- [`-503`](#invalid-content-signature--503)

### ack.get

Get the ACKs and their signatures for an operation.

#### request

HTTP `POST`, `application/json` body.

```json
{
  "id": 12,
  "method": "ack.get",
  "params": {
    "transfer_ids": [
      "6313972fcf4dfede897ed8cd932c579af0eb83c57ff8a8f6def81412ccd4ae0e",
      "ccde6c4ddc6ebc8add7a88f1f917d85f01881a1cd855a9cdaf87d6fa1b3b38c8"
    ],
    "timestamp": "1791367800",
    "signature": "e7e8be3d72de57e64156e95f2631a30a9eaa0e1a98863c2f1512affa5056f7dc9ca86f117e5621a817e6c8aebf1121f584c68bd5e5666aab2e85662632ef8fb6"
  }
}
```

Params:
- `transfer_ids`: the transfer IDs, all belonging to the same operation
- `timestamp`: the current time, within 300 seconds of the proxy clock
- `signature`: the sender signature, made with the key of the `sender_pk`
  registered via [`ack.prepare`](#ackprepare) (see
  [Keys and signatures](#keys-and-signatures))

#### response

```json
{
  "id": 12,
  "result": {
    "acks": {
      "6313972fcf4dfede897ed8cd932c579af0eb83c57ff8a8f6def81412ccd4ae0e": true
    },
    "signatures": {
      "6313972fcf4dfede897ed8cd932c579af0eb83c57ff8a8f6def81412ccd4ae0e": "e7e8be3d72de57e64156e95f2631a30a9eaa0e1a98863c2f1512affa5056f7dc9ca86f117e5621a817e6c8aebf1121f584c68bd5e5666aab2e85662632ef8fb6"
    }
  }
}
```

Result:
- `acks`: a map from transfer ID to its ACK value (`true` for ACK, `false` for
  NACK)
- `signatures`: a map from transfer ID to the recipient signature

Transfers that have not been responded to yet are not included.

Behavior:
- a request with an unknown transfer ID, or with transfer IDs belonging to more
  than one operation, is refused ([`-104`](#operation-not-found--104))
- once the operation has expired, the set of responses is final
- the operation data is kept until the retention grace after the expiration
  has passed

Errors:
- [`-104`](#operation-not-found--104)
- [`-200`](#invalid--200)
- [`-300`](#missing--300)
- [`-502`](#invalid-endpoint-auth-signature--502)

### ack.clean

Remove the signed TXs of an operation whose signed TX has not been released yet
(not all the transfers have been ACKed), so that it can't be released anymore.

#### request

HTTP `POST`, `application/json` body.

```json
{
  "id": 13,
  "method": "ack.clean",
  "params": {
    "transfer_ids": [
      "6313972fcf4dfede897ed8cd932c579af0eb83c57ff8a8f6def81412ccd4ae0e",
      "ccde6c4ddc6ebc8add7a88f1f917d85f01881a1cd855a9cdaf87d6fa1b3b38c8"
    ],
    "signature": "3835b11be35eba3ac2f18624e2667be619d3b65170cb440a53b64ef15e2b864c48768de9fcd7c8fae7c90c0dc3d0a14b3f0f700316e842c31fd288a4a2fea68c"
  }
}
```

Params:
- `transfer_ids`: all the transfer IDs of the operation
- `signature`: the sender signature, made with the key of the `sender_pk`
  registered via [`ack.prepare`](#ackprepare) (see
  [Keys and signatures](#keys-and-signatures))

#### response

When the signed TXs have been removed:
```json
{ "id": 13, "result": true }
```

When there was no signed TX left:
```json
{ "id": 13, "result": false }
```

Behavior:
- the call is refused once all the transfers have been ACKed (the signed TX has
  been released); it's allowed after partial responses, while after a NACK
  there is no signed TX left to remove
- `ack.clean` and [`ack.post`](#ackpost) calls for the same operation are
  processed atomically: either the signed TXs are removed before the last ACK
  is registered, so no TX is released, or the ACK is registered first and
  `ack.clean` is refused
- a request whose transfer IDs aren't exactly all those of a single operation
  is refused ([`-104`](#operation-not-found--104))
- the operation record is kept, so its transfer IDs can't be reused and later
  [`ack.post`](#ackpost) calls don't release any TX

Errors:
- [`-104`](#operation-not-found--104)
- [`-107`](#already-acked--107)
- [`-200`](#invalid--200)
- [`-300`](#missing--300)
- [`-502`](#invalid-endpoint-auth-signature--502)


## Client behavior

The following rules are not enforced by the proxies but are required for
clients to interoperate correctly.

### Sender

The sender:
- uses a fresh key pair for each operation and includes at most one transfer
  per invoice in an operation
- calls [`invoice.check`](#invoicecheck) before building the operation
- derives the witness seal blinding factor (see
  [Witness seal blinding](#witness-seal-blinding))
- sets the [`ack.prepare`](#ackprepare) expiration sufficiently before the
  earliest invoice expiration among the operation recipients, leaving some
  margin to detect the ACKs and broadcast the TX
- calls [`ack.prepare`](#ackprepare) before [`consignment.post`](#consignmentpost)
  (except for donations)
- doesn't broadcast the TX, including for donations, until the calls to
  [`consignment.post`](#consignmentpost) and [`media.post`](#mediapost) have
  succeeded
- doesn't create a transfer after the invoice expiration
- if a transfer cannot be created on a transport endpoint (e.g.
  [`-111`](#invoice-expired--111), [`-112`](#invoice-quota-exceeded--112),
  [`-113`](#transfer-id-deleted--113), [`-115`](#server-draining--115)),
  tries the other transport endpoints listed in the invoice; only if none
  accepts it, fails the operation, calling [`ack.clean`](#ackclean) first for
  non-donation operations; for a donation whose consignment has already been
  posted for another transfer of the operation, it can only fail the operation
  after spending one of the TX inputs, otherwise it completes it and hands the
  missing consignment over by other means
- verifies the ACK signatures returned by [`ack.get`](#ackget) and saves them,
  together with the invoices, as proof of acceptance
- keeps the consignment, so it can be handed over by other means if needed

Failing an operation:
- before its expiration, an operation can only be failed after a successful
  [`ack.clean`](#ackclean)
- after its expiration (plus some margin for clock skew), the sender must call
  [`ack.get`](#ackget), within the retention grace:
    - if at least one recipient has not ACKed, the TX was never released and
      the operation can be failed
    - if every recipient has ACKed, the TX was released: before the invoice
      expiration of every recipient, the sender progresses the operation,
      broadcasting the TX; otherwise it doesn't broadcast it and either
      progresses the operation if the TX is found on-chain or fails it after
      spending one of the TX inputs; since the TX was certainly released
      (unlike when the operation is lost or an ACK signature is invalid),
      failing it without invalidating the TX is not allowed, and a sender that
      can't spend one of its inputs keeps the operation pending until the TX
      confirms or becomes invalid

When the sender proxy no longer knows an operation
([`-104`](#operation-not-found--104) for a registered operation), the sender
waits until the expiration plus the sender proxy's `retention_grace_secs`
has passed (plus some margin for clock skew), so that recipients that obtained
the TX (or still can) have time to broadcast it, then progresses the operation
if the TX is found on-chain (mempool included) or fails it, either accepting
the risk that a recipient that obtained the TX broadcasts it later or after
spending one of the TX inputs.

When an ACK signature returned by [`ack.get`](#ackget) is invalid, the sender
doesn't trust the ACK nor broadcasts the TX, skips [`ack.clean`](#ackclean) and
handles the operation as when the sender proxy no longer knows it, as the
signed TX may have already been released.

### Recipient

Definition:
- the broadcast deadline (`D`) is the operation `expires_at` returned by
  [`ack.post`](#ackpost) plus the sender proxy's `retention_grace_secs`
- the confirmation margin is an additional waiting time after `D`, chosen by
  the client (e.g. one day), that gives a TX broadcast shortly before `D` time
  to be confirmed and also covers clock differences

The recipient:
- uses a fresh key pair for each invoice and reserves the invoice on all the
  transport endpoints listed in it before sharing it (see [Invoice](#invoice))
- discovers the transfers via [`transfer.list`](#transferlist) and retrieves
  them before the end of the retention grace following the invoice expiration
- fails a transfer whose data the proxy deleted before it was downloaded or,
  for a donation, whose TX has already been broadcast, obtains the consignment
  from the sender by other means
- verifies that the transfer ID matches the one derived from the invoice ID and
  the `sender_pk` returned by [`consignment.get`](#consignmentget)
- keeps, for each invoice, the sender pubkeys of the accepted transfers and
  refuses a transfer with an already seen sender pubkey unless it has the same
  consignment hash (i.e. it's a retry of the same transfer)
- refuses a transfer whose allocation paying the invoice (operation ID and
  assignment index) has already been credited for a different transfer ID
- for witness invoices, identifies the output paying the invoice by its script,
  vout and derived blinding factor, refusing the transfer if no such output
  exists
- for blind invoices, identifies the allocation paying the invoice by the
  invoice concealed seal and keeps the blinded UTXO unspent until the invoice
  expiration has passed (plus some margin for clock skew) and, on each
  transport endpoint, either a [`transfer.list`](#transferlist) made after that
  point shows all transfers retrieved and processed (a
  [`-101`](#invoice-not-reserved--101) at that point can't distinguish a
  deletion with nothing pending from a data wipe, so the recipient chooses
  whether to treat it as nothing pending or to wait for that proxy's retention
  grace) or that proxy's retention grace has passed, and all the accepted
  transfers have been settled or failed; donations count as settled once their
  TX confirms (the recipient broadcasts it itself after downloading the
  transfer) and as failed only once their TX can no longer confirm; a
  non-donation transfer downloaded after the blinded UTXO has been spent must
  be refused (NACK), while a donation's assets are lost if its TX confirms
- once the data of a transfer has been durably saved, calls
  [`transfer.retrieved`](#transferretrieved), or
  [`transfer.delete`](#transferdelete) to also free its quota slot (e.g. for
  refused or fake transfers)
- before posting an ACK, reads `retention_grace_secs` from the sender
  proxy's [`server.info`](#serverinfo) and posts a NACK instead if it's below
  its acceptable minimum
- posts a signed ACK or NACK via [`ack.post`](#ackpost) to the sender transport
  endpoint (except for donations) and, as soon as the signed TX is returned,
  decrypts, verifies and broadcasts it, saving it for later re-broadcasts
- rebroadcasts the signed TX, if needed, only until the broadcast deadline `D`:
  then, if the TX gets confirmed by `D` plus the confirmation margin, it
  progresses the transfer, otherwise it fails it
- periodically verifies, until the invoice expiration, that its reservations
  are still in place on all the transport endpoints, reserving the invoice
  again after a proxy data wipe
- takes into account that, when the invoice lists more than one transport
  endpoint, the total number of transfers can exceed the quota, since each
  proxy enforces it independently; extra non-donation transfers can be
  refused, while a strict global limit requires listing a single transport
  endpoint

After a successful ACK without the signed TX (e.g. a batch operation not yet
ACKed by every recipient, or a non-compliant proxy), the transfer waits for the
broadcast and the recipient posts the same ACK again until:
- the signed TX is returned: it broadcasts it and progresses the transfer
- [`-105`](#operation-expired--105) is returned: the TX has never been released
  and never will be, so it fails the transfer
- [`-104`](#operation-not-found--104) is returned, or no definite answer is
  obtained by `D`: it waits until `D` plus the confirmation margin, then
  progresses the transfer if the TX has been confirmed, otherwise it fails it

A recipient may fail a non-donation transfer before posting its ACK, typically
by posting a NACK.

Before failing a transfer, both parties check whether its TX is on-chain and,
if so, progress it instead; once failed, a transfer is final.


## Protocol errors

Errors follow [JSON-RPC 2.0] rules and their codes are divided in 5 categories:
[State](#state), [Invalid](#invalid), [Missing](#missing),
[Not found](#not-found) and [Security](#security).

When present, `error.data` contains the list of failing fields (`fields`) or,
for [`-114`](#invalid-proof-of-work--114), the current difficulty
(`pow_difficulty`).

The `fields` are parameter names: for values nested in a parameter (entries of
`transfer_data`, items of `transfer_ids`) the inner name is used (e.g.
`recipient_pk`, `transfer_id`), while `file` refers to the uploaded file (e.g.
a different file in [`-100`](#cannot-change--100)).

### State

#### Cannot change (-100)

```json
{
  "id": 1,
  "error": {
    "code": -100,
    "message": "Cannot change",
    "data": {
      "fields": ["expires_at"]
    }
  }
}
```

#### Invoice not reserved (-101)

```json
{ "id": 1, "error": { "code": -101, "message": "Invoice not reserved" } }
```

#### Transfer ID in use (-102)

```json
{ "id": 1, "error": { "code": -102, "message": "Transfer ID in use" } }
```

#### Operation not found (-104)

```json
{ "id": 1, "error": { "code": -104, "message": "Operation not found" } }
```

#### Operation expired (-105)

```json
{ "id": 1, "error": { "code": -105, "message": "Operation expired" } }
```

#### Already acked (-107)

```json
{ "id": 1, "error": { "code": -107, "message": "Already acked" } }
```

#### File too large (-108)

```json
{ "id": 1, "error": { "code": -108, "message": "File too large" } }
```

#### Too many transfers (-109)

```json
{ "id": 1, "error": { "code": -109, "message": "Too many transfers" } }
```

#### Signed TX too large (-110)

```json
{ "id": 1, "error": { "code": -110, "message": "Signed TX too large" } }
```

#### Invoice expired (-111)

```json
{ "id": 1, "error": { "code": -111, "message": "Invoice expired" } }
```

#### Invoice quota exceeded (-112)

```json
{ "id": 1, "error": { "code": -112, "message": "Invoice quota exceeded" } }
```

#### Transfer ID deleted (-113)

```json
{ "id": 1, "error": { "code": -113, "message": "Transfer ID deleted" } }
```

#### Invalid proof of work (-114)

```json
{
  "id": 1,
  "error": {
    "code": -114,
    "message": "Invalid proof of work",
    "data": {
      "pow_difficulty": "18"
    }
  }
}
```

#### Server draining (-115)

The proxy doesn't accept new transfers, other calls keep working.

```json
{ "id": 1, "error": { "code": -115, "message": "Server draining" } }
```

### Invalid

#### Invalid (-200)

```json
{
  "id": 1,
  "error": {
    "code": -200,
    "message": "Invalid",
    "data": {
      "fields": ["invoice_id", "timestamp"]
    }
  }
}
```

### Missing

#### Missing (-300)

```json
{
  "id": 1,
  "error": {
    "code": -300,
    "message": "Missing",
    "data": {
      "fields": ["signature"]
    }
  }
}
```

#### Missing file (-301)

```json
{ "id": 1, "error": { "code": -301, "message": "Missing file" } }
```

### Not found

#### Consignment not found (-400)

```json
{ "id": 1, "error": { "code": -400, "message": "Consignment not found" } }
```

#### Media not found (-401)

```json
{ "id": 1, "error": { "code": -401, "message": "Media not found" } }
```

#### Transfer not found (-402)

```json
{ "id": 1, "error": { "code": -402, "message": "Transfer not found" } }
```

#### Transfer data retrieved (-403)

```json
{ "id": 1, "error": { "code": -403, "message": "Transfer data retrieved" } }
```

### Security

#### Invalid proxy auth signature (-501)

```json
{
  "id": 1,
  "error": {
    "code": -501,
    "message": "Invalid proxy auth signature"
  }
}
```

#### Invalid endpoint auth signature (-502)

```json
{
  "id": 1,
  "error": {
    "code": -502,
    "message": "Invalid endpoint auth signature"
  }
}
```

#### Invalid content signature (-503)

```json
{ "id": 1, "error": { "code": -503, "message": "Invalid content signature" } }
```


## Known implementations

- [rgb-proxy-server](https://github.com/RGB-Tools/rgb-proxy-server)


[JSON-RPC 2.0]: https://www.jsonrpc.org/specification
[Storm protocol]: https://github.com/Storm-WG/storm-spec
[v1.0 changes specification]: v1.0-changes-spec.md
