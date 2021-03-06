---
EEP: ...
title: EOSIO Signing Requests
author: Aaron Cox (@aaroncox), Johan Nordberg (@jnordberg)
revision: 2
status: Draft
type: Standards Track
category: Interface
created: 2019-05-19
updated: 2020-01-16
---

# EOSIO Signing Request

**Table of Contents**

- [Summary](#Summary)
- [Abstract](#Abstract)
- [Motivation](#Motivation)
- [Specification](#Specification)
- [Backwards Compatibility](#Backwards-Compatibility)
- [Use Cases](#Use-Cases)
- [Test Cases](#Test-Cases)
- [Implementations](#Implementations)
- [Appendix](#Appendix)
  - [Base64u](#Base64u)
  - [Chain Aliases](#Chain-Aliases)
  - [Compression](#Compression)
  - [Signing Request - Placeholders](#Signing-Request---Placeholders)     
  - [Signing Request - Schema](#Signing-Request---Schema)
    - [EOSIO ABI](#Signing-Request-represented-as-a-EOSIO-C-struct)
    - [EOSIO C++ struct](#Signing-Request-represented-as-an-EOSIO-ABI)
  - [Signing Request - Templating](#Signing-Request---Templating)
- [Change Log](#Change-Log)
- [Acknowledgements](#Acknowledgements)
- [Copyright](#Copyright)

## Summary

A standard for an EOSIO-based signing request payload to allow communication between applications and signature providers.

## Abstract

EOSIO Signing Requests encapsulate transaction data for transport within multiple mediums (e.g. QR codes and hyperlinks), providing a simple cross-application signaling method between very loosely coupled applications. A standardized request data payload allows instant invocation of specific transaction templates within the user's preferred EOSIO signature provider.

## Motivation

The ability to represent a transaction in a standardized signing request format has been a major factor in driving end user adoption within many blockchain ecosystems. Introducing a similar mechanism into the EOSIO ecosystem would speed up adoption by providing a versatile data format which allows requests across any medium.

While other protocols already exist within EOSIO for more intricate cross-application communication - this proposal seeks to establish a primitive request payload for use in any type of application.

## Specification

The following specification sets out to define the technical standards used and the actual composition of an EOSIO Signing Request. While this written specification centers around JavaScript/JSON, the concepts are be compatible within any modern programming environment.

**Table of Contents - Specification**
- [EOSIO Client Implementation Guidelines](#eosio-client-implementation-guidelines)
- [Signing Request](#signing-request-specification)
  - [Data Format](#data-format)
  - [Header](#header)
  - [Payload](#payload)
    - [`req`](#req)
    - [`broadcast`](#broadcast)
    - [`callback`](#callback)
    - [`chain_id`](#chain_id)


### EOSIO Client Implementation Guidelines

The following are a set of guidelines in which end user applications (e.g. signature providers) that handle EOSIO Signing Requests should respect.

- EOSIO clients **MUST NOT** automatically act upon the data contained within a Signing Request without the user's authorization.
- EOSIO clients **MUST** decode and present the transaction data in a human readable format for review before creating a signature.
- EOSIO clients **SHOULD** inform the user of any callback which will be executed upon completion of a signing request.
- EOSIO clients **SHOULD** register themselves as the handler for the `esr:` URI scheme by default, so long as no other handlers already exist. If a registered handler already exists, they **MAY** prompt the user to change it upon request.
- EOSIO clients **SHOULD** allow the use of proxies when handling callbacks ([Callback Proxies](#callback-proxies)).
- EOSIO clients **SHOULD** use the recommended MIME type when applicable in their implementations.

### Signing Request Specification

In its encapsulated form, and EOSIO Signing Request is a data structure which has been converted to [base64u](#Base64u) (URL safe) and then [compressed](#Compression), and is representable as a string:

```
gmNgZGRkAIFXBqEFopc6760yugsVYWCA0YIwxgKjuxLSL6-mgmQA
```

The above payload is a signing request for a transaction to perform the `voteproducer` action on the `eosio` contract. The data contained within the action itself also specifies the proxy of `greymassvote`.

Once decoded/inflated ([preview decoded payload](https://greymass.github.io/eosio-uri-builder/gmNgZGRkAIFXBqEFopc6760yugsVYWCA0YIwxgKjuxLSL6-mgmQA)) it will return the following SigningRequest:

```
{
  version: 2,
  data: {
    chain_id: [ 'chain_alias', 1 ],
    req: [
      'action[]',
      [
        {
          account: 'eosio',
          name: 'voteproducer',
          authorization: [ { actor: '............1', permission: '............2' } ],
          data: '0100000000000000A032DD181BE9D56500'
        }
      ]
    ],
    flags: 1,
    callback: '',
    info: []
  },
  textEncoder: TextEncoder { encoding: 'utf-8' },
  textDecoder: TextDecoder { encoding: 'utf-8', fatal: false, ignoreBOM: false },
  zlib: {
    deflateRaw: [Function: deflateRaw],
    inflateRaw: [Function: inflateRaw]
  },
  abiProvider: { getAbi: [AsyncFunction: getAbi] },
  signature: undefined
}
```

This SigningRequest can then be used to prompt action from an end user in their preferred signature provider. The signature provider can then resolve the transaction and complete any required templating, sign the transaction, and optionally trigger a callback and/or broadcast the signed transaction to the blockchain.

### Data Format

The decoded payload of each Signing Request consists of two parts:

  - a 1-byte header
  - a X-byte payload (`X = length(path) - 1`)

```
header  request
1000001000000000000000000000...
```

#### Header

The header consists of the first 8 bits, with the first 7 bits representing the protocol version and the last bit denoting if the data is compressed. The protocol version this document describes is `2`, making the only valid headers:

- `0x02` a uncompressed payload
- `0x82` a compressed payload

#### Payload

All data beyond the first 8 bits forms the representation of the SigningRequest. This structure is as follows:

  param            | description
 ------------------|-------------
  `abiProvider`    | A method to provide raw ABI data (when needed)
  `data`           | The signing request data (see below)
  `data.callback`  | A templatable URL a POST request should be triggered to after the transaction is broadcast/signed
  `data.chain_id`  | 32-byte id of target chain or 1-byte alias ([Chain Aliases](#chain-aliases))
  `data.flags`     | Various flags for how to process this transaction
  `data.info`      | Optional metadata to pass along with the request
  `data.req`       | The action, list of actions, or full transaction that should be signed
  `signature`      | An array containing existing signatures if any were provided
  `textEncoder`    | The text encoder used for processing
  `textDecoder`    | The text decoder used for processing
  `version`        | The protocol version of this signing request

Many of these fields are further outlined below. An extended schema of this payload can be found in the Appendix as both an [EOSIO C++ struct](#signing-request-represented-as-a-eosio-c-struct) and an [ABI of ESR](#signing-request-represented-as-an-eosio-abi).

---

##### `data.req`

The actual EOSIO transaction(s) involved in a signing request exist within the `data.req` parameter. This data consists of an array where the first value is the `type` of request and the second value is the `data`.

The `type` of data can be one of the following:

  - `action`: a request containing data for a single EOSIO contract action
  - `action[]`: a request containing a array of actions for one or more EOSIO contract actions
  - `identity`: a request type used to facilitate identity requests (login, etc)
  - `transaction`: a request containing a full EOSIO transaction

Example data structures of each are listed below.

###### `type: 'action'` (a single contract action)

The most basic form of a signing request is to pass a single contract action and the data to include when creating a signature. In simple terms this data can be illustrated as:

```
[
  'action',
  { ... action }
]
```

The first value (type) being `action` indicates that the second value is an object containing a singular action that needs to be both templated and included in a transaction using current TAPoS values.

Example:

```
{
  req: [
    'action',
    {
      account: 'eosio.forum',
      name: 'vote',
      authorization: [ { actor: '............1', permission: '............2' } ],
      data: '0100000000000000000000204643BABA0100'
    }
  ],
}
```

###### `type: 'action[]'` (multiple actions)

The second option is to pass an array of actions, which can all be bundled together into a single transaction. The simple syntax using this method is as follows:

```
[
  'action[]',
  [
    <action1>,
    <action2>,
    ...
  ]
]
```

The first value (type) being `action[]` indicates that the second value is an array of multiple actions that needs to be both templated and included in a transaction using current TAPoS values.

Example:

```
{
  req: [
    'action[]',
    [
      {
        account: 'eosio.token',
        name: 'transfer',
        authorization: [ { actor: '............1', permission: '............2' } ],
        data: { ... action data }
      },
      {
        account: 'eosio.token',
        name: 'transfer',
        authorization: [ { actor: '............1', permission: '............2' } ],
        data: { ... action data }
      }
    ]
  ],
}
```

###### `type: 'transaction'` (full transaction)

The most complex option is to pass a nearly complete transaction (including TAPoS values) directly to the EOSIO client, which will only require minor templating and the addition of a signature. This option is primarily to allow for flexibility, but most use cases should likely use the `action` or `action[]` method to avoid having to deal with the complication of building full transactions.

**Note**: By going this route, the EOSIO client will either have to respect the `expiration` and TAPoS values or alter them. By using the `transaction` method, URIs may have a limited shelf life which can make statically sharing URIs more difficult.

```
[
  'transaction',
  { ... transaction }
]
```

Example:

```
{
  req: [
    'transaction',
    {
      "transaction_id": "bc655c082a2f5738ef8c40ee676daca8f20b2a7fcce7532e92a4613ed342a49c",
      "broadcast": false,
      "transaction": {
        "compression": "none",
        "transaction": {
          "expiration": "2019-03-05T23:01:09",
          "ref_block_num": 15729,
          "ref_block_prefix": 3775151106,
          "max_net_usage_words": 0,
          "max_cpu_usage_ms": 0,
          "delay_sec": 0,
          "context_free_actions": [],
          "actions": [
            {
              "account": "eosio.forum",
              "name": "vote",
              "authorization": [
                {
                  "actor": "............1",
                  "permission": "............2"
                }
              ],
              "data": "0100000000000000000000204643BABA0100"
            }
          ],
          "transaction_extensions": []
        },
        "signatures": []
      }
    }
  ],
}
```

---

##### `data.broadcast`

Each signing request has a boolean field for whether or not the signed transaction should be broadcast to the associated blockchain after a signature has been created.

By default, the `broadcast` parameter is set to `true`.

Setting the `broadcast` field to `false` and specifying a `callback` will allow the signature provider to create a signature for the transaction but prevent the signature provider from broadcasting the transaction to the blockchain. The signature (and additional data) instead will then be passed to the `callback` URL, allowing the originator of the request to finalize the transaction and broadcast if needed.

---

##### `data.callback`

An optional parameter of the signing request is the `callback`, which when set indicates how an EOSIO client should proceed after the transaction has completed. The `callback` itself is a string containing a full URL. This URL will be triggered after the transaction has been either signed or broadcast based on the `flags` provided.

The `flags` value dictates the behavior of EOSIO client, indicating whether it should trigger the callback in the native OS handler (e.g. opening a web browser for `http` or `https`) or perform it in the background. If set to `true` and the URL protocol is either `http` or `https`, EOSIO clients should `POST` to the URL instead of redirecting/opening it in a web browser. For other protocols background behavior is up to the implementer.

The callback URL also includes simple templating with some response parameters. The templating format syntax is `{{param_name}}`, e.g.:

- `https://myapp.com/wallet?tx={{tx}}&included_in={{bn}}`
- `mymobileapp://signed/{{sig}}`

Available Parameters:

  - `bn`: Block number hint (only present if transaction was broadcast).
  - `ex`: Expiration time used when resolving request.
  - `rbn`: Reference block num used when resolving request.
  - `req`: The originating signing request encoded as a uri string.
  - `rid`: Reference block id used when resolving request.
  - `sa`: Signer authority, aka account name.
  - `sp`: Signer permission, e.g. "active".
  - `tx`: Transaction ID as HEX-encoded string.
  - `sig`: The first signature.
  - `sigX`: All signatures are 0-indexed as `sig0`, `sig1`, etc.

---

##### `chain_id`

The `chain_id` parameter accepts two different formats, a [Chain Alias](#chain_alias) or a [Chain ID](#chain_id).

###### Chain Alias

In an effort to maintain a lower payload size, a predefined list of aliases for specific `chain_id` values has been defined and can be specified using an array with `uint8` followed by the ID of the chain the request should use. The following is an example of how to use the `chain_id` with a value of `1`:

```
{
  chain_id: [ 'uint8', 1 ],
  ... signing_request
}
```

A full list of available [Chain Aliases](#chain-aliases) can be found in the Appendix of this document.

###### Chain ID

Alternatively, a 32-byte ID value can be passed as the `chain_id` to specify any specific chain.

```
{
  chain_id: [ 'checksum256', 'aca376f206b8fc25a6ed44dbdc66547c36c6c33e3a119ffbeaef943642f0e906' ],
  ... signing_request
}
```

This is useful for in environments where the aliases may not be available or in chains which may not have an alias.

## Backwards Compatibility

- Revision 2 of the ESR signing protocol introduces breaking changes from Revision 1.

## Use Cases

The EOSIO Signing Request format enables many different methods of communication to convey request data from any application to any signature provider.

The following are a few examples of how these payloads could be transmitted.

### Custom URI Scheme Format

The EOSIO Signing Request in a custom URI scheme format uses the `scheme` and `path` components defined within [RFC 3986](https://www.ietf.org/rfc/rfc3986.txt).

```
esr:<signing_request>
\_/ \_______________/
 |         |
 scheme    path
```

The `scheme` that defines the URI format is `esr`. Any client application capable of handling EOSIO transactions can register itself as the default system handler for this scheme.

The `path` portion of the URI is a represents a "[Signing Request](#signing-request)". The data that makes up each request is serialized using the same binary format as the EOSIO blockchain. Each request is also encoded using a url-safe Base64 variant ([appx: Base64u](#base64u)) and optionally compressed using zlib deflate ([appx: Compression](#compression)).

###### Format Example

The following is an example of a `voteproducer` action on the `eosio` contract within the `path` of the URI.

```
esr:gmNgZGRkAIFXBqEFopc6760yugsVYWCA0YIwxgKjuxLSL6-mgmQA
\_/ \__________________________________________________/
 |         |
 scheme    path
```

Once decoded/inflated ([decode URI payload](https://greymass.github.io/eosio-uri-builder/gmNgZGRkAIFXBqEFopc6760yugsVYWCA0YIwxgKjuxLSL6-mgmQA)) it will return the following signing request:

```
{
  version: 2,
  data: {
    chain_id: [ 'chain_alias', 1 ],
    req: [
      'action[]',
      [
        {
          account: 'eosio',
          name: 'voteproducer',
          authorization: [ { actor: '............1', permission: '............2' } ],
          data: '0100000000000000A032DD181BE9D56500'
        }
      ]
    ],
    flags: 1,
    callback: '',
    info: []
  },
  textEncoder: TextEncoder { encoding: 'utf-8' },
  textDecoder: TextDecoder { encoding: 'utf-8', fatal: false, ignoreBOM: false },
  zlib: {
    deflateRaw: [Function: deflateRaw],
    inflateRaw: [Function: inflateRaw]
  },
  abiProvider: { getAbi: [AsyncFunction: getAbi] },
  signature: undefined
}
```

### URI Usage

Many URI schemes are commonly used within hyperlinks (anchor tags) in HTML and QR codes to allow a camera-based transfer of information in mobile devices. Taking the transaction from the above example of a referendum vote action, with a URI of:

```
esr:gmNgZGRkAIFXBqEFopc6760yugsVYWCA0YIwxgKjuxLSL6-mgmQA
```

The transaction can be triggered with the following examples:

###### Hyperlink

Example:

```
<a href="esr:gmNgZGRkAIFXBqEFopc6760yugsVYWCA0YIwxgKjuxLSL6-mgmQA">
  Clickable Hyperlink
</a>
```

If a user were to click the above link with a EOSIO URI compatible application installed, the transaction would be triggered within the end users chosen EOSIO client.


### Custom QR Code Format

As well as being portable enough for usage within a URI/URL format, the same payload data can also be represented as a QR Code to be consumed by any device with QR scanning capabilities.

![qrcode:gmNgZGRkAIFXBqEFopc6760yugsVYWCA0YIwxgKjuxLSL6-mgmQA](../assets/eep-7/qrcode.png)

Scanning the above QR code on a device with QR Code capabilities could trigger the payload within the end user specified signature provider.

## Test Cases

**Note**: These examples all use [eosjs v20.0.0](https://github.com/EOSIO/eosjs/tree/v20.0.0) for its `Serialize` component.

#### Example - Transaction to encoded PATH

This example will take an EOSIO Signing Request and convert it into a compressed string.

```js
/*
  EOSIO URI Specification

  Example: Encoding an EOSIO Signing Request into a compressed payload
*/

const { Serialize } = require('eosjs');
const zlib = require('zlib');
const util = require('util');

const textEncoder = new util.TextEncoder();
const textDecoder = new util.TextDecoder();

// The signing request to be encoded
const signingRequest = {
  // "chain_id": [ "uint8", 1 ],
  "callback": "https://domain.com",
  "chain_id": [ "chain_id", "aca376f206b8fc25a6ed44dbdc66547c36c6c33e3a119ffbeaef943642f0e906"],
  "flags": 1,
  "info": [],
  "req": [
    "action[]",
    [
      {
        "account": "eosio.forum",
        "name": "vote",
        "authorization": [
          {
            "actor": "............1",
            "permission": "............2"
          }
        ],
        "data": "0100000000000000000000204643BABA0100"
      }
    ]
  ],
}

// The minified ABI struct used to deserialize the request
const abi = {version:"eosio::abi/1.1",types:[{new_type_name:"account_name",type:"name"},{new_type_name:"action_name",type:"name"},{new_type_name:"permission_name",type:"name"},{new_type_name:"chain_alias",type:"uint8"},{new_type_name:"chain_id",type:"checksum256"},{new_type_name:"request_flags",type:"uint8"}],structs:[{name:"permission_level",fields:[{name:"actor",type:"account_name"},{name:"permission",type:"permission_name"}]},{name:"action",fields:[{name:"account",type:"account_name"},{name:"name",type:"action_name"},{name:"authorization",type:"permission_level[]"},{name:"data",type:"bytes"}]},{name:"extension",fields:[{name:"type",type:"uint16"},{name:"data",type:"bytes"}]},{name:"transaction_header",fields:[{name:"expiration",type:"time_point_sec"},{name:"ref_block_num",type:"uint16"},{name:"ref_block_prefix",type:"uint32"},{name:"max_net_usage_words",type:"varuint32"},{name:"max_cpu_usage_ms",type:"uint8"},{name:"delay_sec",type:"varuint32"}]},{name:"transaction",base:"transaction_header",fields:[{name:"context_free_actions",type:"action[]"},{name:"actions",type:"action[]"},{name:"transaction_extensions",type:"extension[]"}]},{name:"info_pair",fields:[{name:"key",type:"string"},{name:"value",type:"bytes"}]},{name:"signing_request",fields:[{name:"chain_id",type:"variant_id"},{name:"req",type:"variant_req"},{name:"flags",type:"request_flags"},{name:"callback",type:"string"},{name:"info",type:"info_pair[]"}]},{name:"identity",fields:[{name:"permission",type:"permission_level?"}]},{name:"request_signature",fields:[{name:"signer",type:"name"},{name:"signature",type:"signature"}]}],variants:[{name:"variant_id",types:["chain_alias","chain_id"]},{name:"variant_req",types:["action","action[]","transaction","identity"]}],actions:[{name:"identity",type:"identity"}]};;

/**
* ------------------------------------------------
* Base64u encoding - URL-Safe Base64 variant no padding.
* Based on https://gist.github.com/jonleighton/958841
* ------------------------------------------------
*/

const charset = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_';

function encode(data) {
    const byteLength = data.byteLength;
    const byteRemainder = byteLength % 3;
    const mainLength = byteLength - byteRemainder;

    const parts = [];

    let a;
    let b;
    let c;
    let d;
    let chunk;

    // Main loop deals with bytes in chunks of 3
    for (let i = 0; i < mainLength; i += 3) {
        // Combine the three bytes into a single integer
        chunk = (data[i] << 16) | (data[i + 1] << 8) | data[i + 2];

        // Use bitmasks to extract 6-bit segments from the triplet
        a = (chunk & 16515072) >> 18; // 16515072 = (2^6 - 1) << 18
        b = (chunk & 258048) >> 12;   // 258048   = (2^6 - 1) << 12
        c = (chunk & 4032) >> 6;      // 4032     = (2^6 - 1) << 6
        d = chunk & 63;               // 63       =  2^6 - 1

        // Convert the raw binary segments to the appropriate ASCII encoding
        parts.push(charset[a] + charset[b] + charset[c] + charset[d]);
    }

    // Deal with the remaining bytes
    if (byteRemainder === 1) {
        chunk = data[mainLength]

        a = (chunk & 252) >> 2 // 252 = (2^6 - 1) << 2

        // Set the 4 least significant bits to zero
        b = (chunk & 3) << 4 // 3   = 2^2 - 1

        parts.push(charset[a] + charset[b])
    } else if (byteRemainder === 2) {
        chunk = (data[mainLength] << 8) | data[mainLength + 1]

        a = (chunk & 64512) >> 10 // 64512 = (2^6 - 1) << 10
        b = (chunk & 1008) >> 4   // 1008  = (2^6 - 1) << 4

        // Set the 2 least significant bits to zero
        c = (chunk & 15) << 2 // 15    = 2^4 - 1

        parts.push(charset[a] + charset[b] + charset[c])
    }

    return parts.join('')
}

const buffer = new Serialize.SerialBuffer({
    textEncoder: textEncoder,
    textDecoder: textDecoder,
})

const requestTypes = Serialize.getTypesFromAbi(Serialize.createInitialTypes(), abi);
const requestAbi = requestTypes.get('signing_request');
requestAbi.serialize(buffer, signingRequest);

let header = 2;
header |= 1 << 7;

const array = new Uint8Array(zlib.deflateRawSync(Buffer.from(buffer.asUint8Array())));

// Build the array containing the header as the first byte followed by the request
const data = new Uint8Array(array.byteLength + 1);
data[0] = header;
data.set(array, 1);

// base64u encode the array to a string
const encoded = encode(data);
console.log(encoded);

/* Output:

gmNcs7jsE9uOP6rL3rrcvpMWUmN27LCdleD836_eTzFz-vCSjZGRYcm-EsZXBqEMILDA6C5QBAKYoLQQTAAIFNycd-1iZGAUyigpKSi20tdPyc9NzMzTS87PZQAA

*/
````

#### Example - Encoded PATH to Transaction

This example will take a compressed payload string and convert it into an EOSIO Signing Request structure.

```js
/*
  EOSIO URI Specification

  Example: Decoding and inflating a encoded payload string
*/

const { Serialize } = require('eosjs');
const zlib = require('zlib');
const util = require('util');

const textEncoder = new util.TextEncoder();
const textDecoder = new util.TextDecoder();

// The URI path to be decoded
const uriPath = 'gmNcs7jsE9uOP6rL3rrcvpMWUmN27LCdleD836_eTzFz-vCSjZGRYcm-EsZXBqEMILDA6C5QBAKYoLQQTAAIFNycd-1iZGAUyigpKSi20tdPyc9NzMzTS87PZQAA';

// The minified ABI struct used to deserialize the request
const abi = {version:"eosio::abi/1.1",types:[{new_type_name:"account_name",type:"name"},{new_type_name:"action_name",type:"name"},{new_type_name:"permission_name",type:"name"},{new_type_name:"chain_alias",type:"uint8"},{new_type_name:"chain_id",type:"checksum256"},{new_type_name:"request_flags",type:"uint8"}],structs:[{name:"permission_level",fields:[{name:"actor",type:"account_name"},{name:"permission",type:"permission_name"}]},{name:"action",fields:[{name:"account",type:"account_name"},{name:"name",type:"action_name"},{name:"authorization",type:"permission_level[]"},{name:"data",type:"bytes"}]},{name:"extension",fields:[{name:"type",type:"uint16"},{name:"data",type:"bytes"}]},{name:"transaction_header",fields:[{name:"expiration",type:"time_point_sec"},{name:"ref_block_num",type:"uint16"},{name:"ref_block_prefix",type:"uint32"},{name:"max_net_usage_words",type:"varuint32"},{name:"max_cpu_usage_ms",type:"uint8"},{name:"delay_sec",type:"varuint32"}]},{name:"transaction",base:"transaction_header",fields:[{name:"context_free_actions",type:"action[]"},{name:"actions",type:"action[]"},{name:"transaction_extensions",type:"extension[]"}]},{name:"info_pair",fields:[{name:"key",type:"string"},{name:"value",type:"bytes"}]},{name:"signing_request",fields:[{name:"chain_id",type:"variant_id"},{name:"req",type:"variant_req"},{name:"flags",type:"request_flags"},{name:"callback",type:"string"},{name:"info",type:"info_pair[]"}]},{name:"identity",fields:[{name:"permission",type:"permission_level?"}]},{name:"request_signature",fields:[{name:"signer",type:"name"},{name:"signature",type:"signature"}]}],variants:[{name:"variant_id",types:["chain_alias","chain_id"]},{name:"variant_req",types:["action","action[]","transaction","identity"]}],actions:[{name:"identity",type:"identity"}]};;

/**
* Base64u - URL-Safe Base64 variant no padding.
* Based on https://gist.github.com/jonleighton/958841
*/

const charset = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_';
const lookup = new Uint8Array(256);
for (let i = 0; i < 64; i += 1) { lookup[charset.charCodeAt(i)] = i; }

function decode(input) {
  const byteLength = input.length * 0.75;
  const data = new Uint8Array(byteLength);

  let a;
  let b;
  let c;
  let d;
  let p = 0;

  for (let i = 0; i < input.length; i += 4) {
    a = lookup[input.charCodeAt(i)];
    b = lookup[input.charCodeAt(i + 1)];
    c = lookup[input.charCodeAt(i + 2)];
    d = lookup[input.charCodeAt(i + 3)];

    data[p++] = (a << 2) | (b >> 4);
    data[p++] = ((b & 15) << 4) | (c >> 2);
    data[p++] = ((c & 3) << 6) | (d & 63);
  }

  return data;
}

// Decode the URI Path string into a Uint8Array byte array
const data = decode(uriPath);

// Retrieve header byte and check protocol version
const header = data[0];
const version = header & ~(1 << 7);
if (version !== 2) {
  throw new Error('Invalid protocol version');
}

// Disregard data beyond header byte
let array = data.slice(1);

// Determine via header if zlib deflated and inflate if needed
if ((header & 1 << 7) !== 0) {
  array = new Uint8Array(zlib.inflateRawSync(Buffer.from(array)));
}

// Create buffer based on the decoded/decompressed byte array
const buffer = new Serialize.SerialBuffer({ textEncoder, textDecoder, array });

// Create and retrieve the signing_request abi
const requestTypes = Serialize.getTypesFromAbi(Serialize.createInitialTypes(), abi);
const requestAbi = requestTypes.get('signing_request');

// Deserialize the buffer using the signing_request abi and return request object
const signingRequest = requestAbi.deserialize(buffer);
console.log(util.inspect(signingRequest, { showHidden: false, depth: null }));

/* Output:

{
  chain_id: [
    'chain_id',
    'ACA376F206B8FC25A6ED44DBDC66547C36C6C33E3A119FFBEAEF943642F0E906'
  ],
  req: [
    'action[]',
    [
      {
        account: 'eosio.forum',
        name: 'vote',
        authorization: [ { actor: '............1', permission: '............2' } ],
        data: '0100000000000000000000204643BABA0100'
      }
    ]
  ],
  flags: 1,
  callback: 'https://domain.com',
  info: []
}

*/
````

## Implementations

Existing implementation of the EOSIO URI Scheme (REV 2) include:

##### JS Libraries
 - [greymass/eosio-signing-request](https://github.com/greymass/eosio-signing-request) ([npm](https://www.npmjs.com/package/eosio-signing-request)): A JavaScript library to facilitate the creation and consumption of EOSIO Signing Requests.

##### Signature Providers
- [Anchor](https://github.com/greymass/eos-voter): ESR compatible wallet/signature provider (formerly "eos-voter", or "Greymass Wallet").

##### User Interfaces
- [EOSIO.to](https://eosio.to) (([src](https://github.com/greymass/eosio.to)): Provides Signing Request verification and signature provider hooks.
- [EOSIO URI Builder](https://greymass.github.io/eosio-uri-builder/) ([src](https://github.com/greymass/eosio-uri-builder)): User Interface to encode/decode EOSIO Signing Requests.

## Appendix

##### Base64u

An URL-safe version of Base64 where `+` is replaced by `-`, `/` by `_` and the padding (`=`) is trimmed.

```
base64
SGn+dGhlcmUh/k5pY2X+b2b+eW91/nRv/mRlY29kZf5tZf46KQ==

base64u
SGn-dGhlcmUh_k5pY2X-b2b-eW91_nRv_mRlY29kZf5tZf46KQ
```

##### Callback Proxies

In an effort to help protect the privacy of EOSIO account holders, EOSIO clients which handle signing requests should allow a configurable proxy service to further anonymize outgoing callbacks.

When a callback is made directly from a signing application, the IP address and other identifiable information is sent along with that request could be potentially used in malicious ways. To prevent this, the use of a simple proxy/forwarder can be implemented within the EOSIO client.

For example, if a signing request specified a callback of:

```
https://example.com/signup?tx=ef82d7c2b81675554a4b58586dcf18c2a03a96ff3b8b408c50b34ed9380f94f5
```

This URL can be URI Encoded and passed to a trusted/no-log 3rd party service in order to forward the information. If a proxy service resided at `https://eosuriproxy.com/redirect/{{URL}}`, the callback URL could be then passed through the service as such:

```js
const proxyUri = `https://eosuriproxy.com/redirect/${encodeURIComponent(https://example.com/signup?tx=ef82d7c2b81675554a4b58586dcf18c2a03a96ff3b8b408c50b34ed9380f94f5)}`
```

The proxy service would intercept the callback and forward the request onto the destination URL.

##### Chain Aliases

The following aliases are predefined:

  value  | name     | chain_id
 --------|----------|----------
  `0x00` | RESERVED |
  `0x01` | EOS      | `aca376f206b8fc25a6ed44dbdc66547c36c6c33e3a119ffbeaef943642f0e906`
  `0x02` | TELOS    | `4667b205c6838ef70ff7988f6e8257e8be0e1284a2f59699054a018f743b1d11`
  `0x03` | JUNGLE   | `038f4b0fc8ff18a4f0842a8f0564611f6e96e8535901dd45e43ac8691a1c4dca`
  `0x04` | KYLIN    | `5fff1dae8dc8e2fc4d5b23b2c7665c97f9e9d8edf2b6485a86ba311c25639191`
  `0x05` | WORBLI   | `73647cde120091e0a4b85bced2f3cfdb3041e266cbbe95cee59b73235a1b3b6f`
  `0x06` | BOS      | `d5a3d18fbb3c084e3b1f3fa98c21014b5f3db536cc15d08f9f6479517c6a3d86`
  `0x07` | MEETONE  | `cfe6486a83bad4962f232d48003b1824ab5665c36778141034d75e57b956e422`
  `0x08` | INSIGHTS | `b042025541e25a472bffde2d62edd457b7e70cee943412b1ea0f044f88591664`
  `0x09` | BEOS     | `b912d19a6abd2b1b05611ae5be473355d64d95aeff0c09bedc8c166cd6468fe4`

##### Compression

If the compression bit is set in the header the signing request data is compressed using zlib deflate.

Using compression is recommended since it generates much shorter URIs (and smaller QR codes) but left optional since when used in a contract bandwidth is often cheaper than CPU time.

The following example shows the same signing request, compressed vs uncompressed.

```
original:

esr:AQABAACmgjQD6jBVAAAAVy08zc0BAQAAAAAAAAABAAAAAAAAADEBAAAAAAAAAAAAAAAAAChdoGgGAAAAAAAERU9TAAAAABBzaGFyZSBhbmQgZW5qb3khAQA

zlib deflated:

esr:gWNgZGBY1mTC_MoglIGBIVzX5uxZRqAQGMBoQxgDAjRiF2SwgVksrv7BIFqgOCOxKFUhMS9FITUvK79SkZEBAA
```

##### Signing Request - Placeholders

Within the payload of a signing request, placeholders may be set in both `authorization` and `data` (sub)fields. REV 2 of the URI Specification defines two placeholders:

- `............1` represents the account name used to sign the transaction.
- `............2` represents the account permission used to sign the transaction.

These placeholders should be resolved within EOSIO Clients based on the data provided by the signer when resolving a transaction.

Given the following signing request example:

```js
{ account: "eosio.token",
  name: "transfer",
  authorization: [{actor: "............1", permission: "............2"}],
  data: {
    from: "............1",
    to: "bar",
    quantity: "42.0000 EOS",
    memo: "Don't panic" }}
```

The EOSIO Client handling the request should notice the placeholder values and resolve their values. In this instance, if it were being signed with the authority `foo@active`, the action would resolve to:


```js
{ account: "eosio.token",
  name: "transfer",
  authorization: [{actor: "foo", permission: "active"}],
  data: {
    from: "foo",
    to: "bar",
    quantity: "42.0000 EOS",
    memo: "Don't panic" }}
```

This allows the end user control over which account to trigger an action with and requires no knowledge of the account within a URI.

##### Signing Request - Schema

The data represented in a Signing Request can be represented in the following structures.

###### Signing Request represented as a EOSIO C++ struct:

```cpp
#include <eosiolib/eosio.hpp>
#include <eosiolib/action.hpp>
#include <eosiolib/transaction.hpp>

using namespace eosio;
using namespace std;

typedef checksum256 chain_id;
typedef uint8 chain_alias;

struct callback {
    string url;
    bool background;
};

struct signing_request {
    variant<chain_alias, chain_id> chain_id;
    variant<action, vector<action>, transaction> req;
    bool broadcast;
    optional<callback> callback;
}
```

###### Signing Request represented as an EOSIO ABI:

```json
{
    version: 'eosio::abi/1.1',
    types: [
        {
            new_type_name: 'account_name',
            type: 'name',
        },
        {
            new_type_name: 'action_name',
            type: 'name',
        },
        {
            new_type_name: 'permission_name',
            type: 'name',
        },
        {
            new_type_name: 'chain_alias',
            type: 'uint8',
        },
        {
            new_type_name: 'chain_id',
            type: 'checksum256',
        },
        {
            new_type_name: 'request_flags',
            type: 'uint8',
        },
    ],
    structs: [
        {
            name: 'permission_level',
            fields: [
                {
                    name: 'actor',
                    type: 'account_name',
                },
                {
                    name: 'permission',
                    type: 'permission_name',
                },
            ],
        },
        {
            name: 'action',
            fields: [
                {
                    name: 'account',
                    type: 'account_name',
                },
                {
                    name: 'name',
                    type: 'action_name',
                },
                {
                    name: 'authorization',
                    type: 'permission_level[]',
                },
                {
                    name: 'data',
                    type: 'bytes',
                },
            ],
        },
        {
            name: 'extension',
            fields: [
                {
                    name: 'type',
                    type: 'uint16',
                },
                {
                    name: 'data',
                    type: 'bytes',
                },
            ],
        },
        {
            name: 'transaction_header',
            fields: [
                {
                    name: 'expiration',
                    type: 'time_point_sec',
                },
                {
                    name: 'ref_block_num',
                    type: 'uint16',
                },
                {
                    name: 'ref_block_prefix',
                    type: 'uint32',
                },
                {
                    name: 'max_net_usage_words',
                    type: 'varuint32',
                },
                {
                    name: 'max_cpu_usage_ms',
                    type: 'uint8',
                },
                {
                    name: 'delay_sec',
                    type: 'varuint32',
                },
            ],
        },
        {
            name: 'transaction',
            base: 'transaction_header',
            fields: [
                {
                    name: 'context_free_actions',
                    type: 'action[]',
                },
                {
                    name: 'actions',
                    type: 'action[]',
                },
                {
                    name: 'transaction_extensions',
                    type: 'extension[]',
                },
            ],
        },
        {
            name: 'info_pair',
            fields: [
                {
                    name: 'key',
                    type: 'string',
                },
                {
                    name: 'value',
                    type: 'bytes',
                },
            ],
        },
        {
            name: 'signing_request',
            fields: [
                {
                    name: 'chain_id',
                    type: 'variant_id',
                },
                {
                    name: 'req',
                    type: 'variant_req',
                },
                {
                    name: 'flags',
                    type: 'request_flags',
                },
                {
                    name: 'callback',
                    type: 'string',
                },
                {
                    name: 'info',
                    type: 'info_pair[]',
                },
            ],
        },
        {
            name: 'identity',
            fields: [
                {
                    name: 'permission',
                    type: 'permission_level?',
                },
            ],
        },
        {
            name: 'request_signature',
            fields: [
                {
                    name: 'signer',
                    type: 'name',
                },
                {
                    name: 'signature',
                    type: 'signature',
                },
            ],
        },
    ],
    variants: [
        {
            name: 'variant_id',
            types: ['chain_alias', 'chain_id'],
        },
        {
            name: 'variant_req',
            types: ['action', 'action[]', 'transaction', 'identity'],
        },
    ],
    actions: [
        {
            name: 'identity',
            type: 'identity',
        },
    ],
}
```

## Change Log

- 2020/01/16: Updated to Revision 2.
- 2019/10/28: Added change log, MIME type recommendation

## Acknowledgements

This proposal is inspired by the Bitcoin's [BIP 0021](https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki), Ethereum's [ERC-67](https://github.com/ethereum/EIPs/issues/67), [EIP-681](https://eips.ethereum.org/EIPS/eip-681), [EIP-831](http://eips.ethereum.org/EIPS/eip-831), and Steem's [URI Spec](https://github.com/steemit/steem-uri-spec). Implementations of the URI protocol within these ecosystems pioneered the way by establishing a baseline for future adaptations like this.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
