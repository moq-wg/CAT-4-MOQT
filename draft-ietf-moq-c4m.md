---
title: "Authorization scheme for MOQT using Common Access Tokens"
abbrev: "CAT-4-MOQT"
category: info

docname: draft-ietf-moq-c4m-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: ""
workgroup: "Media Over QUIC"
keyword:
 - media over quic
 - authorization
 - common access token
 - CAT
venue:
  group: "Media Over QUIC"
  type: ""
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "moq-wg/CAT-4-MOQT"
  latest: "https://moq-wg.github.io/CAT-4-MOQT/"


author:

  -
    ins: C. Lemmons
    name: Chris Lemmons
    organization: Comcast
    email: Chris_Lemmons@comcast.com

  -
    ins: G. Simon
    name: Gwendal Simon
    organization: Synamedia
    email: gsimon@synamedia.com

  -
    ins: S. Nandakumar
    name: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com

  -
    ins: W. Law
    name: "Will Law"
    organization: Akamai
    email: wilaw@akamai.com

normative:

  Composite: I-D.draft-lemmons-cose-composite-claims-02
  MoQTransport: I-D.draft-ietf-moq-transport-20
  EDN: I-D.draft-ietf-cbor-edn-literals
  BASE64: RFC4648
  CAT:
    title: "CTA 5007-B Common Access Token"
    date: April 2025
    target: https://shop.cta.tech/products/cta-5007
  DPoP: RFC9449
  JWK-THUMB: RFC7638
  COSE: RFC9052
  COSE-ALG: RFC9053
  CWT: RFC8392
  CWT-CNF: RFC8747
informative:


--- abstract

A token-based authorization scheme for use with Media Over QUIC Transport.


--- middle

# Introduction

This draft introduces a token-based authorization scheme for use with MOQT {{MoQTransport}}.
The scheme protects access to the relay during session establishment and also contrains the
actions which the client may take once connected.

This draft defines version 1 of this specification.

## Overview of the authorization workflow

* An end-user logs-in to a distribution service. The service authenticates the user (via
  username/password, OAuth, 2FA or another method). The methods involved in this authentication step
  lie outside the scope of this draft.
* Based upon the identity and permissions granted to that end-user, the service generates a token. A
  token is a data structure that has been serialized into a byte array. The token encodes information
  such as the user's ID, constraints on how and when they can access the MOQT distribution network and
  contraints on the actions they can take once connected. The token may be signed to make it
  tamper-resistent.
* The token is given in the clear to the end-user, along with a URL to connect to the edge relay of a MOQT
  distribution network. The edge relay is part of a trusted MOQT distribution network. It has previously
  shared secrets with the distribution service, so that this relay is entitled to decrypt related tokens and
  to validate signatures.
* The end-user client application provides the token to the MOQT distribution relay when it connects. This
  connection may be established over WebTransport or raw QUIC.
* The relay decrypts the token upon receipt and validates the signature. Based upon claims conveyed in
  the token, the relay accepts or rejects the connection.
* If the relay accepts the connection, then the client will take a series of MOQT actions: PUBLISH_NAMESPACE,
  SUBSCRIBE_NAMESPACE, SUBSCRIBE or FETCH. For each of these, it will supply the token it received using
  the AUTHENTICATION parameter.
* As an alternative to this workflow, the distribution service may vend multiple tokens to the client. The
  client may use one of those tokens to establish the initial conneciton and others to authorize its actions.

~~~ascii

     End User              Distribution Service         MOQT Relay
        |                         |                         |
        |                         |  0. Share secrets       |
        |                         |<----------------------->|
        |                         |   (offline/pre-setup)   |
        |                         |                         |
        |  1. Login/Authenticate  |                         |
        |<----------------------->|                         |
        |                         |                         |
        |  2. Generate C4M Token  |                         |
        |       + Relay URL       |                         |
        |<------------------------|                         |
        |                         |                         |
        |  3. Connect to Relay with Token                   |
        |-------------------------------------------------->|
        |                         |                         |
        |                         |  4. Validate Token      |
        |                         |<----------------------->|
        |                         | (previously shared      |
        |                         |     secrets)            |
        |                         |                         |
        |  5. Accept/Reject Connection                      |
        |<--------------------------------------------------|
        |                         |                         |
        |  6. MOQT Actions with Token Authorization         |
        |<------------------------------------------------->|
        |     (SETUP, PUBLISH_NAMESPACE, SUBSCRIBE,         |
        |      SUBSCRIBE_NAMESPACE, PUBLISH, FETCH,         |
        |        REQUEST_UPDATE)                            |
        |                         |                         |
        |                         |  7. Revalidate Token    |
        |                         |<----------------------->|
        |                         |   (if moqt-reval set,   |
        |                         |    repeats at interval  |
        |                         |    e.g., every 5 min)   |
~~~

# Token format

This draft uses a single token format, namely the Common Access Token (CAT) {{CAT}}. The token is supplied
as a byte array. When it must be cast to a string for inclusion in a URL, it is Base64 encoded {{BASE64}}.

To provide control over the MOQT actions, this draft defines a new CBOR Web Token (CWT) Claim called "moqt".
Use of the moqt claim is optional for clients. Support for processing the moqt claim is mandatory for relays.

The default for all actions is "Blocked" and this does not need to be communicated in the token.
As soon as a token is provided, all actions are explicitly blocked unless explicitly enabled.

## moqt claim

The "moqt" claim is defined by the following CDDL:

~~~~~~~~~~~~~~~
$$Claims-Set-Claims //= (moqt-label => moqt-value)
moqt-label = TBD_MOQT
moqt-value = { moqt-scopes-key => moqt-scopes, * int => any }
moqt-scopes-key = 0
moqt-scopes = [ + moqt-scope ]
moqt-scope = [ moqt-actions, ? [ + moqt-ns-match ], ? moqt-track-match ]
moqt-actions = [ + moqt-action ]
moqt-action = int
moqt-ns-match = bin-match / nil
moqt-track-match = bin-match

bin-match = bstr / [ match-type, match-value ]
match-type = prefix-match / suffix-match
match-value = bstr

prefix-match = 1
suffix-match = 2
~~~~~~~~~~~~~~~

The "moqt" claim bounds the scope of MOQT actions for which the token can provide
access. The claim value is a CBOR map. Key 0 holds the scopes array. Each scope
is itself an array of one to three elements: the actions (an array of integers), the
namespace matches (an array of match objects), and the track match (a single match
object).

The actions are integers defined as follows:

|----------------------|-----|--------------------------------|
| Action               | Key | Reference                      |
|----------------------|-----|--------------------------------|
| SETUP                |  1  | {{MoQTransport}} Section 10.3  |
| PUBLISH_NAMESPACE    |  2  | {{MoQTransport}} Section 10.16 |
| SUBSCRIBE_NAMESPACE  |  3  | {{MoQTransport}} Section 10.19 |
| SUBSCRIBE            |  4  | {{MoQTransport}} Section 10.7  |
| REQUEST_UPDATE       |  5  | {{MoQTransport}} Section 10.9  |
| PUBLISH              |  6  | {{MoQTransport}} Section 10.11 |
| FETCH                |  7  | {{MoQTransport}} Section 10.13 |
| TRACK_STATUS         |  8  | {{MoQTransport}} Section 10.15 |
| SUBSCRIBE_TRACKS     |  9  | {{MoQTransport}} Section 10.20 |
|----------------------|-----|--------------------------------|

The scope of the moqt claim is limited to the actions provided in the array.
Any action not present in the array is not authorized by moqt claim.

When a match object is a byte string, it is an exact match. When a match object
is an array, the first element is the match type and the second is the match value.

Matches are performed bytewise against the corresponding field of the
Full Track Name (as defined in Section 2.4.1 of {{MoQTransport}}). The
first namespace match object is applied to the first field in the Track Namespace,
and so on. The match for the track name is matched against the Track Name.

Exact matches must match exactly, prefix matches must match the beginning
of the byte string, and suffix matches must match the end of the byte string.

The track namespace match and track name match are optional. If the length of the
scope array is two, then no track name match is performed at all and the scope of
the token includes all track names. If the length is one, the scope includes all
namespaces as well as no matching is performed. The list of actions is mandatory.

A nil match object is special: it only matches the end of the list of namespaces. This
allows the scope to be limited to a precise namespace length. If the list of namespace
match objects does not end with a nil match object, then the scope includes all longer
namespaces that start with fields that match. Note that nil MUST only appear as the
last element in the namespace match array; placing nil elsewhere is invalid.

No normalization is applied to the values against which to match; it is performed bytewise.

### Extensibility

Map keys other than 0 are reserved for future revisions of this
specification. A recipient that finds a key it does not recognize in the
"moqt" claim map MUST treat the claim set containing that claim as not
acceptable. When that claim set is the token's top-level claim set, the
token is rejected. This fail-closed behavior ensures that new keys which
restrict authorization scope cannot be silently skipped.

Issuers that need to offer alternative scope formats can place each in a
separate claim set within an "or" claim {{Composite}}, so recipients that
do not understand one format can still accept another.

### Text examples of permissions to help with CDDL construction

#### Notation Used in Examples

Full Track Names in this draft are represented using Extended Diagnostic Notation (EDN) as defined in {{EDN}} as an array with two elements: an array of namespace fields and a track name.

Example: Allow with an exact match `[['example','com'],'/bob']`

~~~~~~~~~~~~~~~
{
    /moqt/ TBD_MOQT: { 0: [[
        [ /ANNOUNCE/ 2, /SUBSCRIBE_NAMESPACE/ 3, /PUBLISH/ 6, /FETCH/ 7 ],
        ['example','com',nil],
        '/bob'
    ]]}
}
~~~~~~~~~~~~~~~

~~~~~~~~~~~~~~~
Permits
* [['example','com'], '/bob']

Prohibits
* [['example','com'], '']
* [['example','com'], '/bob/123']
* [['example','com'], '/alice']
* [['example','com'], '/bob/logs']
* [['alternate','example','com'], '/bob']
* [['12345'], '']
* [['example'], 'com/bob']
* [['example','com','/bob'], '']
* [['example','com',''], '/bob']
~~~~~~~~~~~~~~~

Example: Allow with a prefix match `[['example','com'],'/bob']`

~~~~~~~~~~~~~~~
{
    /moqt/ TBD_MOQT: { 0: [[
        [ /ANNOUNCE/ 2, /SUBSCRIBE_NAMESPACE/ 3, /PUBLISH/ 6, /FETCH/ 7 ],
        ['example','com',nil],
        [ /prefix/ 1, '/bob']
    ]]}
}
~~~~~~~~~~~~~~~

~~~~~~~~~~~~~~~
Permits
* [['example','com'], '/bob']
* [['example','com'], '/bob/123']
* [['example','com'], '/bob/logs']

Prohibits
* [['example','com'], '']
* [['example','com'], '/alice']
* [['alternate','example','com'], '/bob']
* [['12345'], '']
* [['example'], 'com/bob']
~~~~~~~~~~~~~~~

Example: Allow namespaces starting with `['example','com']` (any length) with exact track name `'/bob'`

~~~~~~~~~~~~~~~
{
    /moqt/ TBD_MOQT: { 0: [[
        [ /PUBLISH_NAMESPACE/ 2, /SUBSCRIBE_NAMESPACE/ 3, /PUBLISH/ 6, /FETCH/ 7 ],
        { /exact/ 0: 'example.com'},
        { /exact/ 0: '/bob'}
    ]]}
}
~~~~~~~~~~~~~~~

~~~~~~~~~~~~~~~
Permits
* [['example','com'], '/bob']
* [['example','com',''], '/bob']
* [['example','com','bob'], '/bob']

Prohibits
* [['example','com'], '']
* [['example','com'], '/bob/123']
* [['example','com'], '/alice']
* [['example','com'], '/bob/logs']
* [['alternate','example','com'], '/bob']
* [['12345'], '']
* [['example'], 'com/bob']
* [['example','com','/bob'], '']
~~~~~~~~~~~~~~~


Example: Allow namespaces starting with `['example','com']` with any track name

~~~~~~~~~~~~~~~
{
    /moqt/ TBD_MOQT: { 0: [[
        [ /PUBLISH_NAMESPACE/ 2, /SUBSCRIBE_NAMESPACE/ 3, /PUBLISH/ 6, /FETCH/ 7 ],
        ['example','com']
    ]]}
}
~~~~~~~~~~~~~~~

~~~~~~~~~~~~~~~
Permits
* [['example','com'], '/bob']
* [['example','com',''], '/bob']
* [['example','com','bob'], '/bob']
* [['example','com'], '']
* [['example','com'], '/bob/123']
* [['example','com'], '/alice']
* [['example','com'], '/bob/logs']
* [['example','com','/bob'], '']

Prohibits
* [['alternate','example','com'], '/bob']
* [['12345'], '']
* [['example'], 'com/bob']
~~~~~~~~~~~~~~~

Example: Allow with a prefix match on an individual namespace element

This example shows how to use a prefix match within a specific namespace field. The second namespace element must start with `'user-'`.

~~~~~~~~~~~~~~~
{
    /moqt/ TBD_MOQT: { 0: [[
        [ /ANNOUNCE/ 2, /SUBSCRIBE_NAMESPACE/ 3, /PUBLISH/ 6, /FETCH/ 7 ],
        ['example', [ /prefix/ 1, 'user-'], nil],
        '/data'
    ]]}
}
~~~~~~~~~~~~~~~

~~~~~~~~~~~~~~~
Permits
* [['example','user-alice'], '/data']
* [['example','user-bob'], '/data']
* [['example','user-'], '/data']

Prohibits
* [['example','alice'], '/data']
* [['example','user-alice','extra'], '/data']
* [['example','USER-alice'], '/data']
* [['example'], '/data']
~~~~~~~~~~~~~~~

Example: Allow with a suffix match on track name

This example demonstrates suffix matching, which matches the end of a byte string.

~~~~~~~~~~~~~~~
{
    /moqt/ TBD_MOQT: { 0: [[
        [ /PUBLISH/ 6 ],
        ['example','com',nil],
        [ /suffix/ 2, '.json']
    ]]}
}
~~~~~~~~~~~~~~~

~~~~~~~~~~~~~~~
Permits
* [['example','com'], 'data.json']
* [['example','com'], '/api/response.json']
* [['example','com'], '.json']

Prohibits
* [['example','com'], 'data.xml']
* [['example','com'], 'json']
* [['example','com'], 'data.JSON']
* [['example','com'], 'data.json.bak']
~~~~~~~~~~~~~~~

### Multiple actions

Multiple actions may be communicated within the same token, with different
permissions. This can be facilitated by the logical claims defined in
{{Composite}} or simply by defining multiple limits,
depending on the required restrictions. In both cases, the order in which
limits are declared and evaluated is unimportant. The evaluation stops after
the first acceptable result is discovered.

#### Example of evaluating multiple actions in the same token:

~~~~~~~~~~~~~~~
{
    /moqt/ TBD_MOQT: { 0: [
        [[/PUBLISH/ 6], ['example','com',nil], [ /prefix/ 1, '/bob']],
        [[/PUBLISH/ 6], ['example','com',nil], '/logs/12345/bob']
    ]},
    /exp/ 4: 1750000000
}
~~~~~~~~~~~~~~~

* (1) PUBLISH (Allow with a prefix match on track name) [['example','com'], '/bob*']
* (2) PUBLISH (Allow with an exact match) [['example','com'], '/logs/12345/bob']

Evaluating `[['example','com'],'/bob/123']` would succeed on test 1 and test 2 would never be evaluated.
Evaluating `[['example','com'],'/logs/12345/bob']` would fail on test 1 but then succeed on test 2.
Evaluating `[['example','com'],'']` would fail on test 1 and on test 2.

In addition, the entire token expires at 2025-05-02T21:57:24+00:00.

#### Example of evaluating multiple actions with related claims:

If there are other claims that depend on which MOQT limit applies, a logical claim is required:

~~~~~~~~~~~~~~~
{
    /or/ TBD_OR: [
        {
            /moqt/ TBD_MOQT: { 0: [[[/PUBLISH/ 6], ['example','com'], [ /prefix/ 1, 'bob']]]},
            /exp/ 4: 1750000000
        },
        {
            /moqt/ TBD_MOQT: { 0: [[[/PUBLISH/ 6], ['example','com'], 'logs/12345/bob']]},
            /exp/ 4: 1750000600
        }
    ]
}
~~~~~~~~~~~~~~~

This provides access to the same tracks as the previous example, but in this
case, the token is valid for publishing logs up to 10 minutes after the time at
which the publishing of the bob track expires.

## moqt-reval claim

The "moqt-reval" claim is defined by the following CDDL:

~~~~~~~~~~~~~~~
$$Claims-Set-Claims //= (moqt-reval-label => moqt-reval-value)
moqt-reval-label = TBD_MOQT_REVAL
moqt-reval-value = number
~~~~~~~~~~~~~~~

The "moqt-reval" claim indicates that the token must be
revalidated for ongoing streams. If the token is no longer acceptable, the
actions authorized by it MUST NOT be permitted to continue.

The "moqt-reval-value" is a revalidation interval, expressed in seconds.
It provides an upper bound on how long a
token may be considered acceptable for an ongoing stream. A revalidator MAY
revalidate sooner.

If the revalidation interval is smaller than the recipient is prepared
or able to revalidate, the recipient MUST reject the token. If a recipient is
unable to revalidate tokens, it MUST reject all tokens with a "moqt-reval"
claim.

A token can be revalidated by simply validating it again, just as if it were
new. However, since some claims, signatures, MACs, and other attributes that
could contribute to unacceptability may be incapable of changing acceptability
in the duration, a revalidator may optimize by skipping some of the checks as
long as the outcome of the validation is the same. Revalidators SHOULD skip
reverifying MACs and signatures when the list of acceptable issuer keys is
unchanged.

When the value of this claim is zero, the token MUST NOT be revalidated. This
is the default behaviour when the claim is not present.

This claim MUST NOT be used outside of a base claimset. If used within a composition
claims, the token is not well-formed.

The claim key for this claim is TBD_MOQT_REVAL and the claim value is a number.
Recipients MUST support this claim. This claim is OPTIONAL for issuers.

# Replay Protection without DPoP

In MOQT, the same token MAY be presented for SETUP and again for
subsequent actions (SUBSCRIBE, PUBLISH, FETCH, etc.) on the same
connection. {{CAT}} defines "catreplay" value 1 as single use, which
would cause these subsequent actions to fail. For MOQT, this
specification therefore interprets "catreplay" value 1 as single
connection: the token may be used for any number of actions, but only
on one connection.

When "catreplay" is 1, a relay binds the token's "cti" to the QUIC or
WebTransport connection on which the token is first accepted. The
relay MUST NOT reject the token solely because it was previously
accepted on that connection. All other validation and authorization
requirements continue to apply. The relay MUST reject the token on
any other connection. A token with "catreplay" 1 that lacks a "cti"
or an "exp" claim MUST be rejected. After the connection terminates,
the relay MUST retain sufficient replay state to prevent reuse for
as long as the token could otherwise be accepted, including any
permitted expiration-time leeway.

When "catreplay" is 2, replay semantics are implementation-defined.

When "catreplay" is absent or 0, no replay protection is applied.

# DPoP for MOQT

This section defines how Demonstrating Proof of Possession (DPoP) {{DPoP}}
is used with CAT tokens in MOQT. DPoP binds a token to a client key pair
so that a stolen token cannot be used without the corresponding private key.

Token acquisition ({{DPoP}} Sections 4, 5, 6, and 8) is unchanged: the
client obtains a DPoP-bound CAT token from the authorization server as
described in RFC 9449. This section adapts the proof-of-possession
side ({{DPoP}} Sections 7 and 9) for MOQT.

MOQT is not HTTP, so the HTTP-specific `htm` and `htu` claims from {{DPoP}}
do not apply. Instead, each DPoP proof carries an Authorization Context
(`actx`) that binds it to a specific MOQT action, namespace, and track.

## Token Binding

The authorization server binds the CAT token to the client's public key
using the "cnf" claim with a "jkt" (JWK Thumbprint {{JWK-THUMB}})
confirmation method {{CWT-CNF}}. The "catdpop" claim ({{CAT}} Section 4.8)
controls proof freshness window and replay settings.

A complete example of a CAT token and its corresponding DPoP proof
is given in {{dpop-example}}.

## DPoP Proof Structure

For each MOQT action the client creates a fresh DPoP proof, a
COSE_Sign1 {{COSE}} signed with the bound private key.

Protected header:

~~~~
{
  / alg: ES256 / 1: -7,
  / typ / 16: "dpop-proof+cwt",
  / COSE_Key / 4: { ... }
}
~~~~

The header MUST contain an asymmetric signature algorithm from
{{COSE-ALG}}, the type string "dpop-proof+cwt", and the client's
public key as a COSE_Key. Symmetric algorithms MUST NOT be used.

The following algorithms are defined for use with MOQT DPoP proofs:

|------------|----------|----------------------------------------|
| Algorithm  | COSE alg | Reference                              |
|------------|----------|----------------------------------------|
| ES256      | -7       | ECDSA w/ SHA-256 (mandatory to support)|
| ES384      | -35      | ECDSA w/ SHA-384                       |
| EdDSA      | -8       | EdDSA (Ed25519 / Ed448)                |
|------------|----------|----------------------------------------|

Relays MUST support ES256. Support for ES384 and EdDSA is RECOMMENDED.

Claims:

~~~~ cddl
dpop-claims = {
  7 => bstr,              ; cti - unique proof identifier
  6 => int,               ; iat - issued-at timestamp
  actx-label => actx,     ; authorization context
  ? ath-label => bstr,    ; SHA-256 hash of the access token
  ? nonce-label => tstr,  ; relay-supplied nonce
}

actx = {
  0: "moqt",              ; type
  1: tstr,                ; action
  2: tstr,                ; tns - track namespace
  ? 3: tstr,              ; tn  - track name
}
~~~~

The `ath` claim, when present, MUST contain the SHA-256 hash of the
CAT token bytes as presented in the AUTHENTICATION parameter. Relays
MUST verify that the hash matches the token accompanying the proof.
The `ath` claim SHOULD be included in all DPoP proofs.

The `nonce` claim carries a relay-supplied nonce for additional replay
protection per {{DPoP}} Section 9. If a relay requires nonces, it
communicates the nonce value in a MOQT protocol parameter (the
mechanism for this is defined by {{MoQTransport}}). When the client
receives a nonce, it MUST include it in the next DPoP proof. Nonce
support is OPTIONAL.

### Namespace and Track Encoding

The `tns` and `tn` fields encode binary MOQT names as text strings
using the serialization format from Section 1.5.1 of {{MoQTransport}}.

The `tns` field serializes each namespace tuple element individually,
then joins them with hyphens (`-`). The `tn` field serializes the
track name the same way (a single element, no hyphens).

Within each element, printable ASCII bytes (0x21-0x7E) except dot (`.`)
and hyphen (`-`) are represented as-is. All other bytes, including dot
and hyphen, are hex-escaped as `.XX` (dot followed by two lowercase
hex digits).

Examples:

- Namespace ("example.com", "app"), track "camera1":
  `"tns": "example.2ecom-app"`, `"tn": "camera1"`
- Namespace ("conference", "room1"), track "audio.opus":
  `"tns": "conference-room1"`, `"tn": "audio.2eopus"`
- Namespace with binary bytes (0xFF, 0x01) and (0x02):
  `"tns": ".ff.01-.02"`

### Action Identifiers

The `actx.action` field uses the following strings:

|----------------------|-------------|
| MOQT Action          | actx.action |
|----------------------|-------------|
| CLIENT_SETUP         | SETUP       |
| SERVER_SETUP         | SETUP       |
| PUBLISH_NAMESPACE    | PUB_NS      |
| SUBSCRIBE_NAMESPACE  | SUB_NS      |
| SUBSCRIBE            | SUBSCRIBE   |
| REQUEST_UPDATE       | REQ_UPDATE  |
| PUBLISH              | PUBLISH     |
| FETCH                | FETCH       |
| TRACK_STATUS         | TRK_STATUS  |
|----------------------|-------------|

### Example {#dpop-example}

The following shows a CAT token and the DPoP proof a client would
create to SUBSCRIBE on namespace ("cdn", "example.com") with a prefix
match on track "/sports/".

CAT token issued by the authorization server:

~~~~
{
  / cnf / 8: {
    / jkt / 3: h'0123...abcdef'
  },
  / moqt / TBD_MOQT: {0: [
    [
      [/SUBSCRIBE/ 4, /FETCH/ 7],
      ['cdn','example.com',nil],
      [ /prefix/ 1, '/sports/']
    ]
  ]},
  / catdpop / 321: {
    0: 300,
    1: 1
  },
  / exp / 4: 1750000000
}
~~~~

DPoP proof created by the client for a SUBSCRIBE to track
"/sports/live":

~~~~
Protected: {
  / alg: ES256 / 1: -7,
  / typ / 16: "dpop-proof+cwt",
  / COSE_Key / 4: {
    1: 2, -1: 1,
    -2: h'...',
    -3: h'...'
  }
}
Claims: {
  / cti / 7: h'a1b2c3d4',
  / iat / 6: 1705123456,
  / actx / TBD: {
    0: "moqt",
    1: "SUBSCRIBE",
    2: "cdn-example.2ecom",
    3: "/sports/live"
  },
  / ath / TBD: h'7d41f23b...'
}
~~~~

The token authorizes SUBSCRIBE and FETCH on namespace
("cdn", "example.com") for tracks matching prefix "/sports/".
The proof binds to the specific SUBSCRIBE action and the target
track "/sports/live", which matches the prefix. The "jkt" in
the token and the COSE_Key in the proof header correspond to the
same key pair.

## Relay Validation

When a relay receives a MOQT action with both a CAT token and a DPoP
proof, it MUST perform the following checks. The relay MUST reject the
request if any check fails.

1. Validate the CAT token (signature, expiration, "moqt" scope).
2. Extract "jkt" from the token's "cnf" claim.
3. Verify the DPoP proof signature using the embedded public key.
4. Verify the algorithm is an asymmetric algorithm from the
   supported set; reject symmetric algorithms.
5. Confirm SHA-256 of the proof's public key matches the "jkt" value.
6. If the "ath" claim is present, verify it matches the SHA-256 hash
   of the accompanying CAT token.
7. Check proof freshness: "iat" MUST fall within the "catdpop" window.
8. If a nonce was issued, verify the "nonce" claim matches.
9. Verify `actx.type` is "moqt".
10. Verify `actx.action` matches the requested MOQT action.
11. Verify `actx.tns` (and `actx.tn` if present) match the target
    namespace and track.
12. If "catdpop" jti setting is 1, check "cti" for replay.

## Security Considerations for DPoP

This section inherits all security considerations from {{DPoP}}.
Additional considerations specific to MOQT are listed below.

DPoP proofs are single-use: each MOQT action MUST carry a fresh proof
with a new "cti" and current "iat". Relays SHOULD reject proofs whose
"iat" falls outside the "catdpop" window.

Symmetric algorithms (HMAC, AES-MAC, etc.) MUST NOT be used for DPoP
proof signatures. Only asymmetric algorithms are permitted.

The `actx.type` field MUST be "moqt". Relays MUST reject proofs with
any other type value to prevent cross-protocol proof reuse.

Clients MUST NOT reuse key pairs across unrelated deployments.

When the `ath` claim is present, relays MUST verify it matches the
SHA-256 hash of the presented CAT token to prevent a DPoP proof
issued for one token from being used with a different token.

Implementations MUST ensure the encoded DPoP proof fits within MOQT
and QUIC transport limits for the AUTHENTICATION parameter.

## Flow

~~~ascii
Client                     Auth Server                Relay
  |                             |                        |
  | 1. Generate key pair        |                        |
  |                             |                        |
  | 2. Auth request + pub key   |                        |
  |----------------------------->                        |
  |                             |                        |
  | 3. CAT (cnf+catdpop+moqt)  |                        |
  |<-----------------------------|                        |
  |                             |                        |
  | 4. MOQT action + CAT + DPoP proof                    |
  |----------------------------------------------------->|
  |                             |  5. Validate CAT       |
  |                             |  6. Validate DPoP      |
  |                             |  7. Authorize action   |
  | 8. Response                 |                        |
  |<-----------------------------------------------------|
~~~

For each subsequent MOQT action, the client creates a fresh DPoP proof
(new cti, current iat) and sends it alongside the same CAT token.

# Adding a token to a URL

Any time an application wishes to add a CAT token to a URL or path element, the token SHOULD first
be Base64 encoded {{BASE64}}. The syntax and method of modifying the URL is left to the application
to define and is not constrained by this specification.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

## Authentication vs Authorization

This specification defines an authorization scheme, not an authentication scheme.
User authentication (verifying the identity of a user via credentials, OAuth, 2FA, etc.)
occurs prior to token issuance and is outside the scope of this document.

The tokens defined in this specification convey authorization - they grant permissions
for specific MOQT actions (such as SUBSCRIBE, PUBLISH, ANNOUNCE) on specific namespaces
and tracks. A valid token does not authenticate a user; rather, it authorizes
the bearer to perform the actions specified in the token's claims.

Implementers should ensure that user authentication is performed by appropriate
mechanisms before tokens are issued. The security of the authorization scheme
depends on the security of the token issuance process, including proper user
authentication.

DPoP security considerations are covered in the "DPoP for MOQT" section.

## Replay Protection for Bearer Tokens

Without DPoP, connection-scoped replay protection limits but does not
eliminate token theft risk — an attacker who intercepts the token
before the client connects can still use it. Deployments that need
stronger guarantees SHOULD use DPoP.

The binding of a token to a connection is state held by the relay that
accepted it. In deployments with multiple relays, such as a CDN or
relay cluster, the same token can be replayed against another relay
unless the relays share this state. Such deployments SHOULD share
"cti" bindings among relays that accept the same tokens, or use DPoP.


# IANA Considerations

IANA will register the following claims in the "CBOR Web Token (CWT) Claims" registry:

|------------------------|----------------|-------------------|
|                        | moqt           | moqt-reval        |
|------------------------|----------------|-------------------|
| Claim Name             | moqt           | moqt-reval        |
| Claim Description      | MOQT Action    | MOQT revalidation |
| JWT Claim Name         | N/A            | N/A               |
| Claim Key              | TBD_MOQT (1+2) | TBD_MOQT (1+2)    |
| Claim Value Type       | map            | number            |
| Change Controller      | IESG           | IESG              |
| Specification Document | RFCXXXX        | RFCXXXX           |
|------------------------|----------------|-------------------|

\[RFC Editor: Please replace RFCXXXX with the published RFC number for this
document.\]

This document also registers the following claims used in DPoP proofs:

|------------------------|---------|---------|---------|
|                        | actx    | ath     | nonce   |
|------------------------|---------|---------|---------|
| Claim Name             | actx    | ath     | nonce   |
| Claim Description      | Authorization Context | Access Token Hash | DPoP Nonce |
| JWT Claim Name         | actx    | ath     | nonce   |
| Claim Key              | TBD     | TBD     | TBD     |
| Claim Value Type       | map     | bstr    | tstr    |
| Change Controller      | IESG    | IESG    | IESG    |
| Specification Document | RFCXXXX | RFCXXXX | RFCXXXX |
|------------------------|---------|---------|---------|

## MOQT Auth Token Type Registry

This document registers the following entry in the "MOQT Auth Token Type"
registry established by {{MoQTransport}}:

|-------------|-------------------|---------------------------|
| Token Type  | Token Name        | Specification             |
|-------------|-------------------|---------------------------|
| 0x01        | CAT               | This document             |
|-------------|-------------------|---------------------------|

### CAT Token Type (0x01)

When the Auth Token Type is set to 0x01, the Token Payload field contains
a Common Access Token (CAT) {{CAT}} serialized as a CBOR-encoded CWT
(CBOR Web Token).

The token MUST be processed according to the validation rules defined in
this specification. Relays receiving a token with this type MUST:

- Validate the token signature or MAC
- Verify token expiration and other standard CWT claims
- Process the "moqt" claim (if present) to authorize MOQT actions
- Process the "moqt-reval" claim (if present) for revalidation requirements
- Process DPoP claims (if present) according to the "DPoP for MOQT" section

If the token fails validation, the relay MUST reject the connection or
action with an appropriate error.

--- back

#  Appendix A: Test Vectors

This appendix provides test vectors in JSON format for cross-implementation
validation of CAT tokens for MOQT. Tokens use COSE_Mac0 (CBOR tag 17) for
HMAC-SHA256 or COSE_Sign1 (CBOR tag 18) for ES256. Token strings are the
base64url encoding of the full COSE structure.

The JSON is only a container for the vectors; it does not define a JSON
representation of CAT claims. The "payload_cbor_hex" value is the
normative claim set. The "claims_edn" value shows the same claim set in
CBOR Extended Diagnostic Notation {{EDN}}, with claim names as comments.

Long string values are folded across lines to fit this document. The line
breaks and the indentation that follows them are not part of the JSON
string. To unfold a hex or base64url value, remove each line break and the
indentation after it. In "description" and "claims_edn" values, replace
each line break and its indentation with a single space.

The moqt and moqt-reval claims do not yet have assigned claim keys
(TBD_MOQT and TBD_MOQT_REVAL). These vectors use the Private Use claim keys
-65537 for moqt and -65538 for moqt-reval {{!RFC8392}} as placeholders.

## Keys

The following keys are used throughout these test vectors:

~~~ json
{
  "es256_private_key":
    "c9afa9d845ba75166b5c215767b1d6934e50c3db36e89b127b8a622b120f6721",
  "es256_public_key_x":
    "60fed4ba255a9d31c961eb74c6356d68c049b8923b61fa6ce669622e60f29fb6",
  "es256_public_key_y":
    "7903fe1008b8bc99a41ae9e95628bc64f2f1b20c2d7e9f5177a3c294d4462299",
  "hmac_sha256":
    "000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f"
}
~~~

## CBOR Encoding of Claims

These vectors validate correct CBOR encoding of individual claim types.

~~~ json
[
  {
    "id": "cbor_issuer_only",
    "description": "Minimal token with only issuer claim",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\"}",
    "payload_cbor_hex":
      "a101781868747470733a2f2f617574682e6578616d706c652e636f6d"
  },
  {
    "id": "cbor_core_claims",
    "description": "All core CWT claims (iss, aud, exp, nbf, cti)",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\",
       3 /aud/: [\"https://relay.example.com\"], 4 /exp/: 1700086400,
       5 /nbf/: 1700000000, 7 /cti/: h'746573742d746f6b656e2d303031'}",
    "payload_cbor_hex":
      "a501781868747470733a2f2f617574682e6578616d706c652e636f6d
       0381781968747470733a2f2f72656c61792e6578616d706c652e636f
       6d041a65554280051a6553f100074e746573742d746f6b656e2d3030
       31"
  },
  {
    "id": "cbor_cat_version_uri",
    "description": "CAT version (uint 1) and URI match rule",
    "claims_edn":
      "{310 /catv/: 1,
       312 /catu/: {1 /host/: {0 /exact/: \"example.com\"}}}",
    "payload_cbor_hex":
      "a219013601190138a101a1006b6578616d706c652e636f6d"
  },
  {
    "id": "cbor_network_identifiers",
    "description": "Network identifiers: IP, CIDR, ASN, ASN range",
    "claims_edn":
      "{311 /catnip/: [52(h'c0a80164'), 52([8, h'0a']), 64512, [64512,
       64768]]}",
    "payload_cbor_hex":
      "a119013784d83444c0a80164d8348208410a19fc008219fc0019fd00"
  },
  {
    "id": "cbor_geographic_claims",
    "description":
      "Geographic claims: coordinates, geohash, ISO 3166, altitude",
    "claims_edn":
      "{282 /geohash/: \"9q8yyk\", 316 /catgeoiso3166/: [\"US\",
       \"CA\"], 317 /catgeocoord/: [[37.7749, -122.4194, 100]],
       318 /catgeoalt/: [10, 5]}",
    "payload_cbor_hex":
      "a419011a6639713879796b19013c8262555362434119013d8183fb40
       42e32fec56d5d0fbc05e9ad77318fc50186419013e820a05"
  },
  {
    "id": "cbor_uri_match_rules",
    "description":
      "URI match rules: host exact, path prefix, extension exact",
    "claims_edn":
      "{312 /catu/: {1 /host/: {0 /exact/: \"example.com\"},
       3 /path/: {1 /prefix/: \"/vod/\"},
       8 /extension/: {0 /exact/: \".m3u8\"}}}",
    "payload_cbor_hex":
      "a1190138a301a1006b6578616d706c652e636f6d03a101652f766f64
       2f08a100652e6d337538"
  },
  {
    "id": "cbor_catu_with_moqt",
    "description":
      "catu (HTTP) and moqt (MOQT) in one token: host
       exact + path prefix, and Subscribe/Fetch on namespace
       (example.com, vod)",
    "claims_edn":
      "{312 /catu/: {1 /host/: {0 /exact/: \"example.com\"},
       3 /path/: {1 /prefix/: \"/vod/\"}}, -65537 /moqt/: {0: [[[4,
       7], [h'6578616d706c652e636f6d', h'766f64']]]}}",
    "payload_cbor_hex":
      "a2190138a201a1006b6578616d706c652e636f6d03a101652f766f64
       2f3a00010000a1008182820407824b6578616d706c652e636f6d4376
       6f64"
  },
  {
    "id": "cbor_alpn",
    "description": "ALPN protocol identifiers",
    "claims_edn":
      "{314 /catalpn/: [h'6d6f712d3030', h'6833']}",
    "payload_cbor_hex":
      "a119013a82466d6f712d3030426833"
  }
]
~~~

## Token Structure

These vectors validate the full COSE_Mac0 (tag 17) and COSE_Sign1 (tag 18)
token structure with cryptographic verification. Each vector includes the
COSE binary encoding (cose_hex) and its base64url representation (cose_b64).

~~~ json
[
  {
    "id": "token_hmac_minimal",
    "description": "Minimal COSE_Mac0 token signed with HMAC-SHA256",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\",
       3 /aud/: [\"https://relay.example.com\"], 4 /exp/: 1700086400}",
    "algorithm": "HMAC-SHA256",
    "algorithm_id": 5,
    "header_cbor_hex": "a201051063434154",
    "payload_cbor_hex":
      "a301781868747470733a2f2f617574682e6578616d706c652e636f6d
       0381781968747470733a2f2f72656c61792e6578616d706c652e636f
       6d041a65554280",
    "tag_hex":
      "16ca16a2d4e2476528ae858d91b00f257a507fe57284bfb13d7d6ca1
       f065e230",
    "key_hex":
      "000102030405060708090a0b0c0d0e0f101112131415161718191a1b
       1c1d1e1f",
    "cose_hex":
      "d18448a201051063434154a0583fa301781868747470733a2f2f6175
       74682e6578616d706c652e636f6d0381781968747470733a2f2f7265
       6c61792e6578616d706c652e636f6d041a65554280582016ca16a2d4
       e2476528ae858d91b00f257a507fe57284bfb13d7d6ca1f065e230",
    "cose_b64":
      "0YRIogEFEGNDQVSgWD-jAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       A4F4GWh0dHBzOi8vcmVsYXkuZXhhbXBsZS5jb20EGmVVQoBYIBbKFqLU
       4kdlKK6FjZGwDyV6UH_lcoS_sT19bKHwZeIw",
    "valid": true
  },
  {
    "id": "token_hmac_full",
    "description":
      "COSE_Mac0 token with core + CAT + informational claims,
       HMAC-SHA256",
    "claims_edn":
      "{1 /iss/: \"https://issuer.moq.example\",
       2 /sub/: \"user:alice@example.com\",
       3 /aud/: [\"https://relay1.example.com\",
       \"https://relay2.example.com\"], 4 /exp/: 1700086400,
       5 /nbf/: 1700000000, 6 /iat/: 1700000000,
       7 /cti/: h'766563746f722d303032', 310 /catv/: 1,
       311 /catnip/: [52(h'cb007132')],
       312 /catu/: {3 /path/: {1 /prefix/: \"/live/\"}}}",
    "algorithm": "HMAC-SHA256",
    "algorithm_id": 5,
    "header_cbor_hex": "a201051063434154",
    "payload_cbor_hex":
      "aa01781a68747470733a2f2f6973737565722e6d6f712e6578616d70
       6c650276757365723a616c696365406578616d706c652e636f6d0382
       781a68747470733a2f2f72656c6179312e6578616d706c652e636f6d
       781a68747470733a2f2f72656c6179322e6578616d706c652e636f6d
       041a65554280051a6553f100061a6553f100074a766563746f722d30
       30321901360119013781d83444cb007132190138a103a101662f6c69
       76652f",
    "tag_hex":
      "29c1a6612ce6d49273fd8c0116b9decbc287662d5ba5d67defafaf6f
       f0c11278",
    "key_hex":
      "000102030405060708090a0b0c0d0e0f101112131415161718191a1b
       1c1d1e1f",
    "cose_hex":
      "d18448a201051063434154a058abaa01781a68747470733a2f2f6973
       737565722e6d6f712e6578616d706c650276757365723a616c696365
       406578616d706c652e636f6d0382781a68747470733a2f2f72656c61
       79312e6578616d706c652e636f6d781a68747470733a2f2f72656c61
       79322e6578616d706c652e636f6d041a65554280051a6553f100061a
       6553f100074a766563746f722d3030321901360119013781d83444cb
       007132190138a103a101662f6c6976652f582029c1a6612ce6d49273
       fd8c0116b9decbc287662d5ba5d67defafaf6ff0c11278",
    "cose_b64":
      "0YRIogEFEGNDQVSgWKuqAXgaaHR0cHM6Ly9pc3N1ZXIubW9xLmV4YW1w
       bGUCdnVzZXI6YWxpY2VAZXhhbXBsZS5jb20DgngaaHR0cHM6Ly9yZWxh
       eTEuZXhhbXBsZS5jb214Gmh0dHBzOi8vcmVsYXkyLmV4YW1wbGUuY29t
       BBplVUKABRplU_EABhplU_EAB0p2ZWN0b3ItMDAyGQE2ARkBN4HYNETL
       AHEyGQE4oQOhAWYvbGl2ZS9YICnBpmEs5tSSc_2MARa53svCh2YtW6XW
       fe-vr2_wwRJ4",
    "valid": true
  },
  {
    "id": "token_es256",
    "description":
      "COSE_Sign1 token signed with ES256
       (P-256 ECDSA, deterministic RFC 6979)",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\",
       3 /aud/: [\"https://moq-relay.example.com\"],
       4 /exp/: 1700086400, 5 /nbf/: 1700000000}",
    "algorithm": "ES256",
    "algorithm_id": -7,
    "header_cbor_hex": "a201261063434154",
    "payload_cbor_hex":
      "a401781868747470733a2f2f617574682e6578616d706c652e636f6d
       0381781d68747470733a2f2f6d6f712d72656c61792e6578616d706c
       652e636f6d041a65554280051a6553f100",
    "signature_hex":
      "ed8cf586f32ba410f2b3097642eb32255c8dc555e9e058c48084809d
       f557887bc5ee4974a9ef5c4557c1557de799628b634cafaa1b74c936
       968e0c9c3a48f222",
    "private_key_hex":
      "c9afa9d845ba75166b5c215767b1d6934e50c3db36e89b127b8a622b
       120f6721",
    "public_key_x_hex":
      "60fed4ba255a9d31c961eb74c6356d68c049b8923b61fa6ce669622e
       60f29fb6",
    "public_key_y_hex":
      "7903fe1008b8bc99a41ae9e95628bc64f2f1b20c2d7e9f5177a3c294
       d4462299",
    "cose_hex":
      "d28448a201261063434154a05849a401781868747470733a2f2f6175
       74682e6578616d706c652e636f6d0381781d68747470733a2f2f6d6f
       712d72656c61792e6578616d706c652e636f6d041a65554280051a65
       53f1005840ed8cf586f32ba410f2b3097642eb32255c8dc555e9e058
       c48084809df557887bc5ee4974a9ef5c4557c1557de799628b634caf
       aa1b74c936968e0c9c3a48f222",
    "cose_b64":
      "0oRIogEmEGNDQVSgWEmkAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       A4F4HWh0dHBzOi8vbW9xLXJlbGF5LmV4YW1wbGUuY29tBBplVUKABRpl
       U_EAWEDtjPWG8yukEPKzCXZC6zIlXI3FVengWMSAhICd9VeIe8XuSXSp
       71xFV8FVfeeZYotjTK-qG3TJNpaODJw6SPIi",
    "valid": true
  }
]
~~~

## DPoP Binding

These vectors validate DPoP (Demonstrating Proof-of-Possession) key binding
in CAT tokens.

~~~ json
[
  {
    "id": "dpop_jwk_binding",
    "description":
      "HMAC token with DPoP cnf-jkt derived from the fixed ES256
       test key (RFC 7638 JWK thumbprint)",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\", 4 /exp/: 1700086400,
       8 /cnf/: {323: h'0cebf1bc9880748a95588905b79843b42ba75cb174055
       e3e246bf87fe00b4a6d'}, 321 /catdpop/: {0: 60, 1: 1}}",
    "dpop": {
      "cnf_jkt_hex":
        "0cebf1bc9880748a95588905b79843b42ba75cb174055e3e246bf87f
         e00b4a6d",
      "cnf_jkt_source":
        "SHA-256(RFC 7638 canonical JWK of ES256 test key)",
      "window_seconds": 60,
      "honor_jti": true
    },
    "payload_cbor_hex":
      "a401781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a6555428008a119014358200cebf1bc9880748a95588905b79843
       b42ba75cb174055e3e246bf87fe00b4a6d190141a200183c0101",
    "cose_hex":
      "d18448a201051063434154a05852a401781868747470733a2f2f6175
       74682e6578616d706c652e636f6d041a6555428008a119014358200c
       ebf1bc9880748a95588905b79843b42ba75cb174055e3e246bf87fe0
       0b4a6d190141a200183c01015820cef11ccef3533e37884638bbff1e
       08cbcf3d6846afcc0257bf2530416b076242",
    "cose_b64":
      "0YRIogEFEGNDQVSgWFKkAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       BBplVUKACKEZAUNYIAzr8byYgHSKlViJBbeYQ7Qrp1yxdAVePiRr-H_g
       C0ptGQFBogAYPAEBWCDO8RzO81M-N4hGOLv_HgjLzz1oRq_MAle_JTBB
       awdiQg"
  },
  {
    "id": "dpop_no_jti",
    "description":
      "DPoP binding with longer window, JTI processing disabled",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\", 4 /exp/: 1700086400,
       8 /cnf/: {323: h'3c82dfd6358ba804bd90879c34e743bbe13aeab798066
       4944f37a0ec0063fe95'}, 321 /catdpop/: {0: 300, 1: 0}}",
    "dpop": {
      "cnf_jkt_hex":
        "3c82dfd6358ba804bd90879c34e743bbe13aeab7980664944f37a0ec
         0063fe95",
      "cnf_jkt_source": "SHA-256 of 'test-public-key-material'",
      "window_seconds": 300,
      "honor_jti": false
    },
    "payload_cbor_hex":
      "a401781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a6555428008a119014358203c82dfd6358ba804bd90879c34e743
       bbe13aeab7980664944f37a0ec0063fe95190141a20019012c0100",
    "cose_hex":
      "d18448a201051063434154a05853a401781868747470733a2f2f6175
       74682e6578616d706c652e636f6d041a6555428008a119014358203c
       82dfd6358ba804bd90879c34e743bbe13aeab7980664944f37a0ec00
       63fe95190141a20019012c01005820ecb7a5b5667b95f20c67341771
       8eac9f5d997fd427485942a61048cdd8c50e0c",
    "cose_b64":
      "0YRIogEFEGNDQVSgWFOkAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       BBplVUKACKEZAUNYIDyC39Y1i6gEvZCHnDTnQ7vhOuq3mAZklE83oOwA
       Y_6VGQFBogAZASwBAFgg7LeltWZ7lfIMZzQXcY6sn12Zf9QnSFlCphBI
       zdjFDgw"
  },
  {
    "id": "dpop_es256_real_binding",
    "description":
      "ES256 token with real JWK thumbprint binding to the
       signing key",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\",
       3 /aud/: [\"https://relay.example.com\"], 4 /exp/: 1700086400,
       8 /cnf/: {323: h'0cebf1bc9880748a95588905b79843b42ba75cb174055
       e3e246bf87fe00b4a6d'}, 321 /catdpop/: {0: 120}}",
    "algorithm": "ES256",
    "jwk_thumbprint_input":
      "{\"crv\":\"P-256\",\"kty\":\"EC\",\"x\":\"YP7UuiVanTHJYet0
       xjVtaMBJuJI7Yfps5mliLmDyn7Y\",\"y\":\"eQP-EAi4vJmkGunpVii
       8ZPLxsgwtfp9Rd6PClNRGIpk\"}",
    "dpop": {
      "cnf_jkt_hex":
        "0cebf1bc9880748a95588905b79843b42ba75cb174055e3e246bf87f
         e00b4a6d",
      "window_seconds": 120,
      "honor_jti": null
    },
    "public_key_x_hex":
      "60fed4ba255a9d31c961eb74c6356d68c049b8923b61fa6ce669622e
       60f29fb6",
    "public_key_y_hex":
      "7903fe1008b8bc99a41ae9e95628bc64f2f1b20c2d7e9f5177a3c294
       d4462299",
    "payload_cbor_hex":
      "a501781868747470733a2f2f617574682e6578616d706c652e636f6d
       0381781968747470733a2f2f72656c61792e6578616d706c652e636f
       6d041a6555428008a119014358200cebf1bc9880748a95588905b798
       43b42ba75cb174055e3e246bf87fe00b4a6d190141a1001878",
    "cose_hex":
      "d28448a201261063434154a0586da501781868747470733a2f2f6175
       74682e6578616d706c652e636f6d0381781968747470733a2f2f7265
       6c61792e6578616d706c652e636f6d041a6555428008a11901435820
       0cebf1bc9880748a95588905b79843b42ba75cb174055e3e246bf87f
       e00b4a6d190141a10018785840cdb69dc3db876b1d05a8f808e5286a
       861026fc8b58b9a9aa97f23bdf0253ff56357f9129ee9b0966770135
       7c8d700721b3029928d4fe573d56a88744dce04f82",
    "cose_b64":
      "0oRIogEmEGNDQVSgWG2lAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       A4F4GWh0dHBzOi8vcmVsYXkuZXhhbXBsZS5jb20EGmVVQoAIoRkBQ1gg
       DOvxvJiAdIqVWIkFt5hDtCunXLF0BV4-JGv4f-ALSm0ZAUGhABh4WEDN
       tp3D24drHQWo-AjlKGqGECb8i1i5qaqX8jvfAlP_VjV_kSnumwlmdwE1
       fI1wByGzApko1P5XPVaoh0Tc4E-C"
  }
]
~~~

## MOQT Authorization Scopes

These vectors validate MOQT scope encoding and authorization matching.
Each vector includes authorization tests that specify expected pass/fail
results for various action, namespace, and track combinations.

~~~ json
[
  {
    "id": "moqt_publisher_exact",
    "description":
      "Publisher scope: exact namespace match, prefix track match",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\", 4 /exp/: 1700086400,
       -65537 /moqt/: {0: [[[2, 6], [h'6578616d706c652e636f6d',
       h'616c696365'], [1, h'766964656f2d']]]}}",
    "moqt_scopes": [
      {
        "actions": [2, 6],
        "action_names": ["PublishNamespace", "Publish"],
        "namespace_matches": [
          {"type": "exact", "pattern_utf8": "example.com",
           "pattern_hex": "6578616d706c652e636f6d"},
          {"type": "exact", "pattern_utf8": "alice",
           "pattern_hex": "616c696365"}
        ],
        "track_match": {
          "type": "prefix", "pattern_utf8": "video-",
          "pattern_hex": "766964656f2d"
        }
      }
    ],
    "payload_cbor_hex":
      "a301781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a655542803a00010000a1008183820206824b6578616d706c652e
       636f6d45616c696365820146766964656f2d",
    "cose_hex":
      "d18448a201051063434154a0584aa301781868747470733a2f2f6175
       74682e6578616d706c652e636f6d041a655542803a00010000a10081
       83820206824b6578616d706c652e636f6d45616c6963658201467669
       64656f2d5820467c57ade72d6be629b745ef33d6e28ec38ba1ca8bd3
       feb851f3832af94fe686",
    "cose_b64":
      "0YRIogEFEGNDQVSgWEqjAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       BBplVUKAOgABAAChAIGDggIGgktleGFtcGxlLmNvbUVhbGljZYIBRnZp
       ZGVvLVggRnxXrecta-Ypt0XvM9bijsOLocqL0_64UfODKvlP5oY",
    "authorization_tests": [
      {"action": 2, "namespace": ["example.com", "alice"],
       "track": "video-hd", "expected": true},
      {"action": 6, "namespace": ["example.com", "alice"],
       "track": "video-sd", "expected": true},
      {"action": 6, "namespace": ["example.com", "alice"],
       "track": "audio-main", "expected": false},
      {"action": 4, "namespace": ["example.com", "alice"],
       "track": "video-hd", "expected": false},
      {"action": 6, "namespace": ["example.com", "bob"],
       "track": "video-hd", "expected": false}
    ]
  },
  {
    "id": "moqt_subscriber_prefix",
    "description":
      "Subscriber scope: prefix namespace match, any track",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\", 4 /exp/: 1700086400,
       -65537 /moqt/: {0: [[[3, 4, 7], [[1,
       h'636f6e666572656e63652e6578616d706c65']]]]}}",
    "moqt_scopes": [
      {
        "actions": [3, 4, 7],
        "action_names": ["SubscribeNamespace", "Subscribe", "Fetch"],
        "namespace_matches": [
          {"type": "prefix",
           "pattern_utf8": "conference.example",
           "pattern_hex": "636f6e666572656e63652e6578616d706c65"}
        ],
        "track_match": null
      }
    ],
    "payload_cbor_hex":
      "a301781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a655542803a00010000a10081828303040781820152636f6e6665
       72656e63652e6578616d706c65",
    "cose_hex":
      "d18448a201051063434154a05845a301781868747470733a2f2f6175
       74682e6578616d706c652e636f6d041a655542803a00010000a10081
       828303040781820152636f6e666572656e63652e6578616d706c6558
       20e0502202b8c01d5ab61508cd19b7dd943b287be0f26c3d37c12e58
       3ca87a937b",
    "cose_b64":
      "0YRIogEFEGNDQVSgWEWjAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       BBplVUKAOgABAAChAIGCgwMEB4GCAVJjb25mZXJlbmNlLmV4YW1wbGVY
       IOBQIgK4wB1athUIzRm33ZQ7KHvg8mw9N8EuWDyoepN7",
    "authorization_tests": [
      {"action": 4, "namespace": ["conference.example.room1"],
       "track": "audio", "expected": true},
      {"action": 7, "namespace": ["conference.example.room2"],
       "track": "video", "expected": true},
      {"action": 4, "namespace": ["other.domain"],
       "track": "audio", "expected": false},
      {"action": 6, "namespace": ["conference.example.room1"],
       "track": "audio", "expected": false}
    ]
  },
  {
    "id": "moqt_multi_scope",
    "description":
      "Multi-scope token: publish to specific namespace, subscribe
       to prefix, with revalidation",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\", 4 /exp/: 1700086400,
       -65537 /moqt/: {0: [[[2, 6], [h'6c6976652e6578616d706c65',
       h'73747564696f2d61']], [[4, 7], [[1,
       h'6c6976652e6578616d706c65']]]]}, -65538 /moqt-reval/: 300}",
    "moqt_reval": 300.0,
    "moqt_scopes": [
      {
        "actions": [2, 6],
        "action_names": ["PublishNamespace", "Publish"],
        "namespace_matches": [
          {"type": "exact", "pattern_utf8": "live.example",
           "pattern_hex": "6c6976652e6578616d706c65"},
          {"type": "exact", "pattern_utf8": "studio-a",
           "pattern_hex": "73747564696f2d61"}
        ],
        "track_match": null
      },
      {
        "actions": [4, 7],
        "action_names": ["Subscribe", "Fetch"],
        "namespace_matches": [
          {"type": "prefix", "pattern_utf8": "live.example",
           "pattern_hex": "6c6976652e6578616d706c65"}
        ],
        "track_match": null
      }
    ],
    "payload_cbor_hex":
      "a401781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a655542803a00010000a1008282820206824c6c6976652e657861
       6d706c654873747564696f2d61828204078182014c6c6976652e6578
       616d706c653a0001000119012c",
    "cose_hex":
      "d18448a201051063434154a05861a401781868747470733a2f2f6175
       74682e6578616d706c652e636f6d041a655542803a00010000a10082
       82820206824c6c6976652e6578616d706c654873747564696f2d6182
       8204078182014c6c6976652e6578616d706c653a0001000119012c58
       20f1080af59a719fbcb7a036aa6f9714f52ada467b8c1677c87ade0f
       cfccaaf50f",
    "cose_b64":
      "0YRIogEFEGNDQVSgWGGkAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       BBplVUKAOgABAAChAIKCggIGgkxsaXZlLmV4YW1wbGVIc3R1ZGlvLWGC
       ggQHgYIBTGxpdmUuZXhhbXBsZToAAQABGQEsWCDxCAr1mnGfvLegNqpv
       lxT1KtpGe4wWd8h63g_PzKr1Dw",
    "authorization_tests": [
      {"action": 6, "namespace": ["live.example", "studio-a"],
       "track": "cam1", "expected": true},
      {"action": 4, "namespace": ["live.example.studio-b"],
       "track": "cam1", "expected": true},
      {"action": 6, "namespace": ["live.example", "studio-b"],
       "track": "cam1", "expected": false},
      {"action": 2, "namespace": ["other.example", "studio-a"],
       "track": "", "expected": false}
    ]
  },
  {
    "id": "moqt_admin_wildcard",
    "description":
      "Admin scope: all actions, no namespace/track restriction",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\", 4 /exp/: 1700086400,
       -65537 /moqt/: {0: [[[0, 1, 2, 3, 4, 5, 6, 7, 8]]]}}",
    "moqt_scopes": [
      {
        "actions": [0, 1, 2, 3, 4, 5, 6, 7, 8],
        "action_names": [
          "ClientSetup", "ServerSetup", "PublishNamespace",
          "SubscribeNamespace", "Subscribe", "RequestUpdate",
          "Publish", "Fetch", "TrackStatus"
        ],
        "namespace_matches": [],
        "track_match": null
      }
    ],
    "payload_cbor_hex":
      "a301781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a655542803a00010000a100818189000102030405060708",
    "cose_hex":
      "d18448a201051063434154a05835a301781868747470733a2f2f6175
       74682e6578616d706c652e636f6d041a655542803a00010000a10081
       81890001020304050607085820172ebe07f227a41c319684ef916fdb
       f142d391fac6902e38bcdcfb3966b46a50",
    "cose_b64":
      "0YRIogEFEGNDQVSgWDWjAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       BBplVUKAOgABAAChAIGBiQABAgMEBQYHCFggFy6-B_InpBwxloTvkW_b
       8ULTkfrGkC44vNz7OWa0alA",
    "authorization_tests": [
      {"action": 0, "namespace": ["any.namespace"],
       "track": "any-track", "expected": true},
      {"action": 6, "namespace": ["any.namespace"],
       "track": "any-track", "expected": true},
      {"action": 8, "namespace": ["any.namespace"],
       "track": "status", "expected": true}
    ]
  },
  {
    "id": "moqt_suffix_match",
    "description":
      "Suffix matching on both namespace and track",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\", 4 /exp/: 1700086400,
       -65537 /moqt/: {0: [[[4], [[2, h'2e6578616d706c652e636f6d']],
       [2, h'2d617564696f']]]}}",
    "moqt_scopes": [
      {
        "actions": [4],
        "action_names": ["Subscribe"],
        "namespace_matches": [
          {"type": "suffix", "pattern_utf8": ".example.com",
           "pattern_hex": "2e6578616d706c652e636f6d"}
        ],
        "track_match": {
          "type": "suffix", "pattern_utf8": "-audio",
          "pattern_hex": "2d617564696f"
        }
      }
    ],
    "payload_cbor_hex":
      "a301781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a655542803a00010000a100818381048182024c2e6578616d706c
       652e636f6d8202462d617564696f",
    "cose_hex":
      "d18448a201051063434154a05846a301781868747470733a2f2f6175
       74682e6578616d706c652e636f6d041a655542803a00010000a10081
       8381048182024c2e6578616d706c652e636f6d8202462d617564696f
       5820602c1ceb2729485a2b5eb09249e254a41d42abd0b63b2846fd67
       7d7fbf593920",
    "cose_b64":
      "0YRIogEFEGNDQVSgWEajAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       BBplVUKAOgABAAChAIGDgQSBggJMLmV4YW1wbGUuY29tggJGLWF1ZGlv
       WCBgLBzrJylIWitesJJJ4lSkHUKr0LY7KEb9Z31_v1k5IA",
    "authorization_tests": [
      {"action": 4, "namespace": ["cdn.example.com"],
       "track": "stream1-audio", "expected": true},
      {"action": 4, "namespace": ["cdn.example.com"],
       "track": "stream1-video", "expected": false},
      {"action": 4, "namespace": ["cdn.other.org"],
       "track": "stream1-audio", "expected": false}
    ]
  }
]
~~~

## Composite Claims

These vectors validate composite claim encoding per
{{Composite}}. Each vector includes an outer
token (iss, exp) with embedded composite claims. Tokens use COSE_Mac0 with
HMAC-SHA256.

The or, nor and and claims do not yet have assigned claim keys. Until they
are allocated, these vectors encode every composite claim key as 0. 0 is
not a valid claim key, so a verifier fails to parse these claims. The
"claims_edn" value and the "operator" field identify which composite claim
each 0 key stands for.

~~~ json
[
  {
    "id": "composite_or_simple",
    "description":
      "OR composite: at least one of two alternative claim sets
       must be acceptable",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\", 4 /exp/: 1700086400,
       0 /or/: [{1 /iss/: \"https://auth.example.com\",
       4 /exp/: 1700086400, 310 /catv/: 1},
       {1 /iss/: \"https://auth-backup.example.com\",
       4 /exp/: 1700090000, 310 /catv/: 1}]}",
    "composite": {"operator": "OR", "claim_key": 0},
    "payload_cbor_hex":
      "a301781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a655542800082a301781868747470733a2f2f617574682e657861
       6d706c652e636f6d041a6555428019013601a301781f68747470733a
       2f2f617574682d6261636b75702e6578616d706c652e636f6d041a65
       55509019013601",
    "cose_hex":
      "d18448a201051063434154a05877a301781868747470733a2f2f6175
       74682e6578616d706c652e636f6d041a655542800082a30178186874
       7470733a2f2f617574682e6578616d706c652e636f6d041a65554280
       19013601a301781f68747470733a2f2f617574682d6261636b75702e
       6578616d706c652e636f6d041a655550901901360158208ebb4ac5a3
       4771116903baff5e544f3047468faeb077820f26ea2df940b6c58d",
    "cose_b64":
      "0YRIogEFEGNDQVSgWHejAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       BBplVUKAAIKjAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29tBBplVUKA
       GQE2AaMBeB9odHRwczovL2F1dGgtYmFja3VwLmV4YW1wbGUuY29tBBpl
       VVCQGQE2AVggjrtKxaNHcRFpA7r_XlRPMEdGj66wd4IPJuot-UC2xY0"
  },
  {
    "id": "composite_and",
    "description":
      "AND composite: both claim sets must be acceptable",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\", 4 /exp/: 1700086400,
       0 /and/: [{4 /exp/: 1700086400, 310 /catv/: 1},
       {4 /exp/: 1700086400, 311 /catnip/: [52(h'0a000000')]}]}",
    "composite": {"operator": "AND", "claim_key": 0},
    "payload_cbor_hex":
      "a301781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a655542800082a2041a6555428019013601a2041a655542801901
       3781d834440a000000",
    "cose_hex":
      "d18448a201051063434154a05841a301781868747470733a2f2f6175
       74682e6578616d706c652e636f6d041a655542800082a2041a655542
       8019013601a2041a6555428019013781d834440a00000058208ca321
       3e71dfe3900fd910fcb7f6d365949e72e81ceba80f2ff24048502e9a
       1a",
    "cose_b64":
      "0YRIogEFEGNDQVSgWEGjAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       BBplVUKAAIKiBBplVUKAGQE2AaIEGmVVQoAZATeB2DRECgAAAFggjKMh
       PnHf45AP2RD8t_bTZZSecugc66gPL_JASFAumho"
  },
  {
    "id": "composite_nor",
    "description":
      "NOR composite: none of the listed claim sets can be
       acceptable",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\", 4 /exp/: 1700086400,
       0 /nor/: [{1 /iss/: \"https://revoked.example.com\",
       4 /exp/: 1700086400}]}",
    "composite": {"operator": "NOR", "claim_key": 0},
    "payload_cbor_hex":
      "a301781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a655542800081a201781b68747470733a2f2f7265766f6b65642e
       6578616d706c652e636f6d041a65554280",
    "cose_hex":
      "d18448a201051063434154a05849a301781868747470733a2f2f6175
       74682e6578616d706c652e636f6d041a655542800081a201781b6874
       7470733a2f2f7265766f6b65642e6578616d706c652e636f6d041a65
       5542805820a5419cd0db084ac9efc26c21ff2663b4ec0cff6622ae48
       719f65738cb93fd19e",
    "cose_b64":
      "0YRIogEFEGNDQVSgWEmjAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       BBplVUKAAIGiAXgbaHR0cHM6Ly9yZXZva2VkLmV4YW1wbGUuY29tBBpl
       VUKAWCClQZzQ2whKye_CbCH_JmO07Az_ZiKuSHGfZXOMuT_Rng"
  },
  {
    "id": "composite_nested",
    "description":
      "Nested composite: OR containing a standalone claim set
       and a nested AND",
    "claims_edn":
      "{1 /iss/: \"https://auth.example.com\", 4 /exp/: 1700086400,
       0 /or/: [{1 /iss/: \"https://primary.example.com\",
       4 /exp/: 1700086400},
       {0 /and/: [{1 /iss/: \"https://secondary.example.com\",
       4 /exp/: 1700086400}, {4 /exp/: 1700086400, 310 /catv/: 1}]}]}",
    "composite": {"operator": "OR", "claim_key": 0},
    "payload_cbor_hex":
      "a301781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a655542800082a201781b68747470733a2f2f7072696d6172792e
       6578616d706c652e636f6d041a65554280a10082a201781d68747470
       733a2f2f7365636f6e646172792e6578616d706c652e636f6d041a65
       554280a2041a6555428019013601",
    "cose_hex":
      "d18448a201051063434154a0587ea301781868747470733a2f2f6175
       74682e6578616d706c652e636f6d041a655542800082a201781b6874
       7470733a2f2f7072696d6172792e6578616d706c652e636f6d041a65
       554280a10082a201781d68747470733a2f2f7365636f6e646172792e
       6578616d706c652e636f6d041a65554280a2041a6555428019013601
       5820c60304bcd67f2f78bf9e83e84fe3a2d13f8e809659a32cc480c7
       6cfe5ae1da06",
    "cose_b64":
      "0YRIogEFEGNDQVSgWH6jAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       BBplVUKAAIKiAXgbaHR0cHM6Ly9wcmltYXJ5LmV4YW1wbGUuY29tBBpl
       VUKAoQCCogF4HWh0dHBzOi8vc2Vjb25kYXJ5LmV4YW1wbGUuY29tBBpl
       VUKAogQaZVVCgBkBNgFYIMYDBLzWfy94v56D6E_jotE_joCWWaMsxIDH
       bP5a4doG"
  }
]
~~~

## Token Validation

These vectors validate token processing: expected pass and fail scenarios.
All tokens use HMAC-SHA256 (algorithm_id 5) with the key specified in the
Keys section unless otherwise noted. Token values are the base64url encoding
of the full COSE_Mac0 structure.

~~~ json
[
  {
    "id": "valid_basic",
    "description":
      "Valid token with correct issuer, audience, and time bounds",
    "token":
      "0YRIogEFEGNDQVSgWEWkAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tA4F4GWh0dHBzOi8vcmVsYXkuZXhhbXBsZS5jb20EGmVVQoAFGm
       VT8QBYIP30vfENhlwaLMuZjMPeYXjz8QWuqCDlyzStGQVQ4_xi",
    "validation": {
      "reference_time": 1700003600,
      "expected_issuers": ["https://auth.example.com"],
      "expected_audiences": ["https://relay.example.com"],
      "expected_result": "valid"
    }
  },
  {
    "id": "invalid_expired",
    "description": "Token with expiration in the past",
    "token":
      "0YRIogEFEGNDQVSgWCKiAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tBBpfXhAAWCDNInUDQ2qsHo8l8Gj0tzb1HIYIpBheVbVoItgT7U
       a6aA",
    "validation": {
      "reference_time": 1700000000,
      "expected_result": "error",
      "expected_error": "TokenExpired"
    }
  },
  {
    "id": "invalid_not_yet_valid",
    "description": "Token with not-before in the future",
    "token":
      "0YRIogEFEGNDQVSgWCijAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tBBplVpQABRplVUKAWCArREpqUH1KiIrMzevyeLr6t_y-L6oheaJ
       GUU9hwhYRbg",
    "validation": {
      "reference_time": 1700000000,
      "expected_result": "error",
      "expected_error": "TokenNotYetValid"
    }
  },
  {
    "id": "invalid_wrong_issuer",
    "description": "Token from untrusted issuer",
    "token":
      "0YRIogEFEGNDQVSgWD-jAXgYaHR0cHM6Ly9ldmlsLmV4YW1wbGUu
       Y29tA4F4GWh0dHBzOi8vcmVsYXkuZXhhbXBsZS5jb20EGmVVQoBYIO
       mQtbjAMxCnCDlMbHiF-XZlN1SV6QSqkZWTguttCpaC",
    "validation": {
      "reference_time": 1700003600,
      "expected_issuers": ["https://auth.example.com"],
      "expected_result": "error",
      "expected_error": "InvalidIssuer"
    }
  },
  {
    "id": "invalid_wrong_audience",
    "description": "Token not intended for this audience",
    "token":
      "0YRIogEFEGNDQVSgWEWjAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tA4F4H2h0dHBzOi8vb3RoZXItcmVsYXkuZXhhbXBsZS5jb20EGm
       VVQoBYIOeCJJJNAkaNHBBkOdcSFVXKNNj0D8GdGkLmOriIB2pX",
    "validation": {
      "reference_time": 1700003600,
      "expected_issuers": ["https://auth.example.com"],
      "expected_audiences": ["https://relay.example.com"],
      "expected_result": "error",
      "expected_error": "InvalidAudience"
    }
  },
  {
    "id": "invalid_tampered_signature",
    "description":
      "COSE_Mac0 token with corrupted tag (first byte flipped)",
    "original_cose_hex":
      "d18448a201051063434154a0583fa301781868747470733a2f2f61757
       4682e6578616d706c652e636f6d0381781968747470733a2f2f72656c
       61792e6578616d706c652e636f6d041a65554280582016ca16a2d4e24
       76528ae858d91b00f257a507fe57284bfb13d7d6ca1f065e230",
    "token":
      "0YRIogEFEGNDQVSgWD-jAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tA4F4GWh0dHBzOi8vcmVsYXkuZXhhbXBsZS5jb20EGmVVQoBYIO
       nKFqLU4kdlKK6FjZGwDyV6UH_lcoS_sT19bKHwZeIw",
    "validation": {
      "expected_result": "error",
      "expected_error": "SignatureVerificationFailed",
      "key_hex":
        "000102030405060708090a0b0c0d0e0f101112131415161718191a1b
         1c1d1e1f"
    }
  },
  {
    "id": "invalid_wrong_key",
    "description": "Token verified with incorrect key",
    "token":
      "0YRIogEFEGNDQVSgWCKiAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tBBplVUKAWCCqD2TUB2jOEalC6kta2ptR7vx6flZ0I9L0RV3MCb
       Z-Aw",
    "validation": {
      "correct_key_hex":
        "000102030405060708090a0b0c0d0e0f101112131415161718191a1b
         1c1d1e1f",
      "wrong_key_hex":
        "ffffffffffffffffffffffffffffffffffffffffffffffffffffffff
         ffffffff",
      "expected_result": "error",
      "expected_error": "SignatureVerificationFailed"
    }
  },
  {
    "id": "invalid_algorithm_mismatch",
    "description":
      "COSE_Mac0 token but verifier expects COSE_Sign1/ES256",
    "token":
      "0YRIogEFEGNDQVSgWCKiAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tBBplVUKAWCCqD2TUB2jOEalC6kta2ptR7vx6flZ0I9L0RV3MCb
       Z-Aw",
    "validation": {
      "token_algorithm_id": 5,
      "verifier_algorithm_id": -7,
      "expected_result": "error",
      "expected_error": "InvalidTokenFormat"
    }
  }
]
~~~

# Acknowledgments
{:numbered="false"}

The IETF moq workgroup
