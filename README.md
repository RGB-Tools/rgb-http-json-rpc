# RGB HTTP JSON-RPC

In order to operate an RGB transfer it is necessary to exchange special files,
called consignments, that represent the offline data used to move asset rights.
Moreover, with NFTs and collectibles there could also be the need to exchange
media files.

While this can be achieved using the [Storm
protocol](https://github.com/Storm-WG/storm-spec), which is a powerful protocol
for distributed storage, it is preferable not to rely on a single
file exchange medium, in order for the RGB ecosystem to be as resilient as
possible.

To provide a simple alternative, we defined this protocol. It can be easily
implemented in almost any programming language and can use any storage system.

The protocol uses a [JSON-RPC 2.0] interface over HTTP(s).
In order to be compliant with the protocol, an implementation must support all
the [protocol methods](#protocol-methods).

The protocol is supported as a consignment endpoint type in the `Invoice`
provided by [rgb-wallet](https://github.com/RGB-WG/rgb-wallet).

Protocol version is `0.2`.


## Protocol methods

### server.info

#### request

```json
{ "id": 0, "method": "server.info" }
```

#### response

```json
{ "id": 0, "result": { "protocol_version": "0.2", "version": "0.2.0", "uptime": 42069 } }
```

### consignment.post

#### request

In addition to the parameters, the consignment file needs to be attached under
the `file` name via multipart form.

```json
{ "id": 9, "method": "consignment.post", "params": { "recipient_id": "txob1nt9ee42dczeu3s0447txykxnqwq2w98ps2c4k8vd9fry903rud7q2wymt7", "txid": "9ccf8491394e6aa2bec99f08ee8a856879505e07791b3aac79f56177808eb49f" } }
```

Params:
- `recipient_id`: the recipient ID for which we want to upload a consignment file
- `txid`: the txid of the related transfer
- `vout`: [optional] the vout for the related transfer

#### response

On first successful upload:
```json
{ "id": 9, "result": true }
```

On successive successful uploads (of the same consignment file):
```json
{ "id": 9, "result": false }
```

Errors:
- [`-101`](#change-uploaded-file-(-101))
- [`-202`](#recipient-id-(-202))
- [`-203`](#txid-(-203))
- [`-204`](#vout-(-204))
- [`-302`](#recipient-id-(-302))
- [`-303`](#file-(-303))
- [`-304`](#txid-(-304))

### consignment.get

#### request

```json
{ "id": 3, "method": "consignment.get", "params": { "recipient_id": "txob1nt9ee42dczeu3s0447txykxnqwq2w98ps2c4k8vd9fry903rud7q2wymt7" } }
```

Params:
- `recipient_id`: the recipient ID for which we want to get a consignment file

#### response

When a consignment is available, it is returned in base64-encoded form:
```json
{ "id": 3, "result": {"consignment": "0f001d0104ffffffff0100f2052a0100000043410496b538e853519c726a2c91e61ec11600ae1390813a627c66fb8be7947be63c52da7589379515d4e0a604f8141781e62294721166bf621e73a82cbf2342c858eeac00000000", "txid": "9ccf8491394e6aa2bec99f08ee8a856879505e07791b3aac79f56177808eb49f"} }
```

Errors:
- [`-202`](#recipient-id-(-202))
- [`-302`](#recipient-id-(-302))
- [`-400`](#consignment-file-(-400))

### media.post

#### request

In addition to the parameters, the media file needs to be attached under
the `file` name via multipart form.

```json
{ "id": 8, "method": "media.post", "params": { "attachment_id": "94667ec87b3c3280b712edba44c7bddb5c23c988dbe3fd40c1079eb913f63bfc" } }
```

Params:
- `attachment_id`: the attachment ID for which we want to upload a media file

#### response

On first successful upload:
```json
{ "id": 8, "result": true }
```

On successive successful uploads (of the same media file):
```json
{ "id": 8, "result": false }
```

Errors:
- [`-101`](#change-uploaded-file-(-101))
- [`-201`](#attachment-id-(-201))
- [`-301`](#attachment-id-(-301))
- [`-303`](#file-(-303))

### media.get

#### request

```json
{ "id": 4, "method": "media.get", "params": { "attachment_id": "94667ec87b3c3280b712edba44c7bddb5c23c988dbe3fd40c1079eb913f63bfc" } }
```

Params:
- `attachment_id`: the attachment ID for which we want to get a media file

#### response

When a media is available, it is returned in base64-encoded form:
```json
{ "id": 4, "result": "0b001dr104fjff7fff0100f2052a0100000043410496b538e853519c726a2c91e61ec11600ae1390813a627c66fb8be7947be63c52da7589379515d4e0a604f8141781e62294721166bf621e73a82cbf2342c858eeac00000000" }
```

Errors:
- [`-201`](#attachment-id-(-201))
- [`-301`](#attachment-id-(-301))
- [`-401`](#media-file-(-401))

### ack.post

#### request

To accept a consignment:

```json
{ "id": 7, "method": "ack.post", "params": {
    "recipient_id": "txob1nt9ee42dczeu3s0447txykxnqwq2w98ps2c4k8vd9fry903rud7q2wymt7",
    "ack": true
} }
```

To reject a consignment:

```json
{ "id": 7, "method": "ack.post", "params": {
    "recipient_id": "txob1nt9ee42dczeu3s0447txykxnqwq2w98ps2c4k8vd9fry903rud7q2wymt7",
    "ack": false
} }
```

Params:
- `recipient_id`: the recipiend ID for which we want to ACK or NACK the associated consignment file
- `ack`: whether the consignment is accepted (`true`) or rejected (`false`)

#### response

On first successful POST:
```json
{ "id": 7, "result": true }
```

On successive successful POSTs (of the same ACK value):
```json
{ "id": 7, "result": false }
```


Errors:
- [`-100`](#change-ack-(-100))
- [`-200`](#ack-(-200))
- [`-202`](#recipient-id-(-202))
- [`-300`](#ack-(-300))
- [`-302`](#recipient-id-(-302))
- [`-400`](#consignment-file-(-400))

### ack.get

#### request

```json
{ "id": 9, "method": "ack.get", "params": { "recipient_id": "txob1nt9ee42dczeu3s0447txykxnqwq2w98ps2c4k8vd9fry903rud7q2wymt7" } }
```

Params:
- `recipient_id`: the recipiend ID for which we want to know if the associated
  consignment file has been ACKed or NACKed

#### response

When counterparty has positively acknowledged (ACK) the consignment:

```json
{ "id": 9, "result": true }
```

When counterparty has negatively acknowledged (NACK) the consignment:

```json
{ "id": 9, "result": false }
```

When counterparty has not (yet) acknowledged the consignment:

```json
{ "id": 9, "result": null }
```

Errors:
- [`-202`](#recipient-id-(-202))
- [`-302`](#recipient-id-(-302))
- [`-400`](#consignment-file-(-400))


## Protocol errors

Errors follow [JSON-RPC 2.0] rules and their codes are divided in 4 categories:
[Cannot](#cannot), [Invalid](#invalid), [Missing](#missing) and
[Not found](#not-found).

In the error examples `<req_params>` (`error.data`) will contain the client
request `params`.

### Cannot

#### Change ACK (-100)

```json
{ "id": 2, "error": "{ \"code\": -100, \"message\": \"Cannot change ACK\", \"data\": <req_params> }" }
```

#### Change uploaded file (-101)

```json
{ "id": 1, "error": "{ \"code\": -101, \"message\": \"Cannot change uploaded file\", \"data\": <req_params> }" }
```

### Invalid

#### Ack (-200)

```json
{ "id": 1, "error": "{ \"code\": -200, \"message\": \"Invalid ACK\", \"data\": <req_params> }" }
```

#### Attachment ID (-201)

```json
{ "id": 1, "error": "{ \"code\": -201, \"message\": \"Invalid attachment ID\", \"data\": <req_params> }" }
```

#### Recipient ID (-202)

```json
{ "id": 1, "error": "{ \"code\": -202, \"message\": \"Invalid recipient ID\", \"data\": <req_params> }" }
```

#### Txid (-203)

```json
{ "id": 1, "error": "{ \"code\": -203, \"message\": \"Invalid TXID\", \"data\": <req_params> }" }
```

#### Vout (-204)

```json
{ "id": 1, "error": "{ \"code\": -204, \"message\": \"Invalid vout\", \"data\": <req_params> }" }
```

### Missing

#### Ack (-300)

```json
{ "id": 1, "error": "{ \"code\": -300, \"message\": \"Missing ACK\", \"data\": <req_params> }" }
```

#### Attachment ID (-301)

```json
{ "id": 1, "error": "{ \"code\": -301, \"message\": \"Missing attachment ID\", \"data\": <req_params> }" }
```

#### Recipient ID (-302)

```json
{ "id": 1, "error": "{ \"code\": -302, \"message\": \"Missing recipient ID\", \"data\": <req_params> }" }
```

#### File (-303)

```json
{ "id": 1, "error": "{ \"code\": -303, \"message\": \"Missing file\", \"data\": <req_params> }" }
```

#### Txid (-304)

```json
{ "id": 1, "error": "{ \"code\": -304, \"message\": \"Missing TXID\", \"data\": <req_params> }" }
```

### Not found

#### Consignment file (-400)

```json
{ "id": 1, "error": "{ \"code\": -400, \"message\": \"Consignment file not found\", \"data\": <req_params> }" }
```

#### Media file (-401)

```json
{ "id": 1, "error": "{ \"code\": -401, \"message\": \"Media file not found\", \"data\": <req_params> }" }
```


## Known implementations

- [rgb-proxy-server](https://github.com/grunch/rgb-proxy-server)


[JSON-RPC 2.0]: https://www.jsonrpc.org/specification
