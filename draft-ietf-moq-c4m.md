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
    ins: W. Law
    name: "Will Law"
    organization: Akamai
    email: wilaw@akamai.com

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
moqt-value = [ + moqt-scope ]
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
access. It is an array of action scopes. Each scope is an array with three
elements: an array of integers that identifies the actions, an array of match objects for
the namespace, and a match object for the track name.

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

When a match object is a byte string, it is an exact match. When a match object is an array, the first element is the match type and the second is the match value.

Matches are performed bytewise against the corresponding field of the Full Track Name (as defined in Section 2.4.1 of {{MoQTransport}}). The first namespace match object is applied to the first field in the Track Namespace, and so on. The match for the track name is matched against the Track Name.

Exact matches must match exactly, prefix matches must match the beginning of the byte string, and suffix matches must match the end of the byte string.

The track namespace match and track name match are optional. If the length of the scope array is two, then no track name match is performed at all and the scope of the token includes all track names. If the length is one, the scope includes all namespaces as well as no matching is performed. The list of actions is mandatory.

A nil match object is special: it only matches the end of the list of namespaces. This allows the scope to be limited to a precise namespace length. If the list of namespace match objects does not end with a nil match object, then the scope includes all longer namespaces that start with fields that match. Note that nil MUST only appear as the last element in the namespace match array; placing nil elsewhere is invalid.

No normalization is applied to the values against which to match; it is performed bytewise.

### Text examples of permissions to help with CDDL construction

#### Notation Used in Examples

Full Track Names in this draft are represented using Extended Diagnostic Notation (EDN) as defined in {{EDN}} as an array with two elements: an array of namespace fields and a track name.

Example: Allow with an exact match `[['example','com'],'/bob']`

~~~~~~~~~~~~~~~
{
    /moqt/ TBD_MOQT: [[
        [ /ANNOUNCE/ 2, /SUBSCRIBE_NAMESPACE/ 3, /PUBLISH/ 6, /FETCH/ 7 ],
        ['example','com',nil],
        '/bob'
    ]]
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
    /moqt/ TBD_MOQT: [[
        [ /ANNOUNCE/ 2, /SUBSCRIBE_NAMESPACE/ 3, /PUBLISH/ 6, /FETCH/ 7 ],
        ['example','com',nil],
        [ /prefix/ 1, '/bob']
    ]]
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
    /moqt/ TBD_MOQT: [[
        [ /PUBLISH_NAMESPACE/ 2, /SUBSCRIBE_NAMESPACE/ 3, /PUBLISH/ 6, /FETCH/ 7 ],
        { /exact/ 0: 'example.com'},
        { /exact/ 0: '/bob'}
    ]]
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
    /moqt/ TBD_MOQT: [[
        [ /PUBLISH_NAMESPACE/ 2, /SUBSCRIBE_NAMESPACE/ 3, /PUBLISH/ 6, /FETCH/ 7 ],
        ['example','com']
    ]]
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
    /moqt/ TBD_MOQT: [[
        [ /ANNOUNCE/ 2, /SUBSCRIBE_NAMESPACE/ 3, /PUBLISH/ 6, /FETCH/ 7 ],
        ['example', [ /prefix/ 1, 'user-'], nil],
        '/data'
    ]]
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
    /moqt/ TBD_MOQT: [[
        [ /PUBLISH/ 6 ],
        ['example','com',nil],
        [ /suffix/ 2, '.json']
    ]]
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
    /moqt/ TBD_MOQT: [
        [[/PUBLISH/ 6], ['example','com',nil], [ /prefix/ 1, '/bob']],
        [[/PUBLISH/ 6], ['example','com',nil], '/logs/12345/bob']
    ],
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
            /moqt/ TBD_MOQT: [[[/PUBLISH/ 6], ['example','com'], [ /prefix/ 1, 'bob']]],
            /exp/ 4: 1750000000
        },
        {
            /moqt/ TBD_MOQT: [[[/PUBLISH/ 6], ['example','com'], 'logs/12345/bob']],
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


# IANA Considerations

IANA will register the following claims in the "CBOR Web Token (CWT) Claims" registry:

|------------------------|----------------|-------------------|
|                        | moqt           | moqt-reval        |
|------------------------|----------------|-------------------|
| Claim Name             | moqt           | moqt-reval        |
| Claim Description      | MOQT Action    | MOQT revalidation |
| JWT Claim Name         | N/A            | N/A               |
| Claim Key              | TBD_MOQT (1+2) | TBD_MOQT (1+2)    |
| Claim Value Type       | array          | number            |
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
validation of CAT tokens for MOQT. Token strings use the base64url encoding
defined in {{BASE64}} with the three-part structure: header.payload.signature.

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
    "claims": {
      "iss": "https://auth.example.com"
    },
    "payload_cbor_hex":
      "a101781868747470733a2f2f617574682e6578616d706c652e636f6d"
  },
  {
    "id": "cbor_core_claims",
    "description": "All core CWT claims (iss, aud, exp, nbf, cti)",
    "claims": {
      "iss": "https://auth.example.com",
      "aud": ["https://relay.example.com"],
      "exp": 1700086400,
      "nbf": 1700000000,
      "cti": "test-token-001"
    },
    "payload_cbor_hex":
      "a501781868747470733a2f2f617574682e6578616d706c652e636f6d03
       81781968747470733a2f2f72656c61792e6578616d706c652e636f6d04
       1a65554280051a6553f100074e746573742d746f6b656e2d303031"
  },
  {
    "id": "cbor_cat_version_usage",
    "description": "CAT version string and usage limit",
    "claims": {
      "catv": "CAT-v1",
      "catu": 5
    },
    "payload_cbor_hex":
      "a2190136664341542d763119013805"
  },
  {
    "id": "cbor_network_identifiers",
    "description": "Network identifiers: IP, CIDR, ASN, ASN range",
    "claims": {
      "catnip": [
        {"type": "ip_address", "value": "192.168.1.100"},
        {"type": "ip_range", "value": "10.0.0.0/8"},
        {"type": "asn", "value": 64512},
        {"type": "asn_range", "value": [64512, 64768]}
      ]
    },
    "payload_cbor_hex":
      "a1190137846d3139322e3136382e312e313030a16869705f72616e6765
       6a31302e302e302e302f38a16361736e19fc00a16961736e5f72616e67
       658219fc0019fd00"
  },
  {
    "id": "cbor_geographic_claims",
    "description": "Geographic claims: coordinates, geohash, ISO 3166, altitude",
    "claims": {
      "geohash": "9q8yyk",
      "catgeoiso3166": ["US", "CA"],
      "catgeocoord": {"lat": 37.7749, "lon": -122.4194, "accuracy": 100.0},
      "catgeoalt": 10
    },
    "payload_cbor_hex":
      "a419011a6639713879796b19013c8262555362434119013da3636c6174
       fb4042e32fec56d5d0636c6f6efbc05e9ad77318fc506861636375726
       16379f956401901 3e0a"
  },
  {
    "id": "cbor_uri_patterns",
    "description": "URI patterns: exact, prefix, suffix",
    "claims": {
      "cath": [
        {"type": "exact", "value": "https://example.com/live/stream1"},
        {"type": "prefix", "value": "https://example.com/vod/"},
        {"type": "suffix", "value": ".m3u8"}
      ]
    },
    "payload_cbor_hex":
      "a119013b83782068747470733a2f2f6578616d706c652e636f6d2f6c6
       976652f73747265616d31a166707265666978781868747470733a2f2f65
       78616d706c652e636f6d2f766f642fa166737566666978652e6d337538"
  },
  {
    "id": "cbor_alpn",
    "description": "ALPN protocol identifiers",
    "claims": {
      "catalpn": ["moq-00", "h3"]
    },
    "payload_cbor_hex":
      "a119013a82666d6f712d3030626833"
  }
]
~~~

## Token Structure

These vectors validate the full token structure (header.payload.signature)
with cryptographic verification.

~~~ json
[
  {
    "id": "token_hmac_minimal",
    "description": "Minimal token signed with HMAC-SHA256",
    "algorithm": "HMAC-SHA256",
    "algorithm_id": -4,
    "claims": {
      "iss": "https://auth.example.com",
      "aud": ["https://relay.example.com"],
      "exp": 1700086400
    },
    "header_cbor_hex": "a201231063434154",
    "header_b64": "ogEjEGNDQVQ",
    "payload_cbor_hex":
      "a301781868747470733a2f2f617574682e6578616d706c652e636f6d
       0381781968747470733a2f2f72656c61792e6578616d706c652e636f
       6d041a65554280",
    "payload_b64":
      "owF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQOBeBlodHRwczovL3
       JlbGF5LmV4YW1wbGUuY29tBBplVUKA",
    "signature_hex":
      "5b5ec60fb1a3f81d18b5e8d7edf4702e55261248def8c13cd6809cf6
       865a6986",
    "signature_b64":
      "W17GD7Gj-B0YtejX7fRwLlUmEkje-ME81oCc9oZaaYY",
    "key_hex":
      "000102030405060708090a0b0c0d0e0f101112131415161718191a1b
       1c1d1e1f",
    "token":
      "ogEjEGNDQVQ.owF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQOBeB
       lodHRwczovL3JlbGF5LmV4YW1wbGUuY29tBBplVUKA.W17GD7Gj-B0Y
       tejX7fRwLlUmEkje-ME81oCc9oZaaYY",
    "valid": true
  },
  {
    "id": "token_hmac_full",
    "description":
      "Token with core + CAT + informational claims, HMAC-SHA256",
    "algorithm": "HMAC-SHA256",
    "algorithm_id": -4,
    "claims": {
      "iss": "https://issuer.moq.example",
      "sub": "user:alice@example.com",
      "aud": [
        "https://relay1.example.com",
        "https://relay2.example.com"
      ],
      "exp": 1700086400,
      "nbf": 1700000000,
      "iat": 1700000000,
      "cti": "vector-002",
      "catv": "CAT-v1",
      "catnip": [{"type": "ip_address", "value": "203.0.113.50"}],
      "catu": 10
    },
    "header_cbor_hex": "a201231063434154",
    "header_b64": "ogEjEGNDQVQ",
    "payload_cbor_hex":
      "aa01781a68747470733a2f2f6973737565722e6d6f712e6578616d70
       6c650276757365723a616c696365406578616d706c652e636f6d0382
       781a68747470733a2f2f72656c6179312e6578616d706c652e636f6d
       781a68747470733a2f2f72656c6179322e6578616d706c652e636f6d
       041a65554280051a6553f100061a6553f100074a766563746f722d30
       3032190136664341542d7631190137816c3230332e302e3131332e35
       301901380a",
    "payload_b64":
      "qgF4Gmh0dHBzOi8vaXNzdWVyLm1vcS5leGFtcGxlAnZ1c2VyOmFsa
       WNlQGV4YW1wbGUuY29tA4J4Gmh0dHBzOi8vcmVsYXkxLmV4YW1wbG
       UuY29teBpodHRwczovL3JlbGF5Mi5leGFtcGxlLmNvbQQaZVVCgAUa
       ZVPxAAYaZVPxAAdKdmVjdG9yLTAwMhkBNmZDQVQtdjEZATeBbDIwMy
       4wLjExMy41MBkBOAo",
    "signature_hex":
      "02aa58a31e34ab53fab3c755b47cf08f458a3603da4d933d7c0b1ce4
       614f44da",
    "signature_b64":
      "AqpYox40q1P6s8dVtHzwj0WKNgPaTZM9fAsc5GFPRNo",
    "key_hex":
      "000102030405060708090a0b0c0d0e0f101112131415161718191a1b
       1c1d1e1f",
    "token":
      "ogEjEGNDQVQ.qgF4Gmh0dHBzOi8vaXNzdWVyLm1vcS5leGFtcGxlAn
       Z1c2VyOmFsaWNlQGV4YW1wbGUuY29tA4J4Gmh0dHBzOi8vcmVsYXkx
       LmV4YW1wbGUuY29teBpodHRwczovL3JlbGF5Mi5leGFtcGxlLmNvbQQ
       aZVVCgAUaZVPxAAYaZVPxAAdKdmVjdG9yLTAwMhkBNmZDQVQtdjEZAT
       eBbDIwMy4wLjExMy41MBkBOAo.AqpYox40q1P6s8dVtHzwj0WKNgPaTZ
       M9fAsc5GFPRNo",
    "valid": true
  },
  {
    "id": "token_es256",
    "description":
      "Token signed with ES256 (P-256 ECDSA, deterministic RFC 6979)",
    "algorithm": "ES256",
    "algorithm_id": -7,
    "claims": {
      "iss": "https://auth.example.com",
      "aud": ["https://moq-relay.example.com"],
      "exp": 1700086400,
      "nbf": 1700000000
    },
    "header_cbor_hex": "a201261063434154",
    "header_b64": "ogEmEGNDQVQ",
    "payload_cbor_hex":
      "a401781868747470733a2f2f617574682e6578616d706c652e636f6d
       0381781d68747470733a2f2f6d6f712d72656c61792e6578616d706c
       652e636f6d041a65554280051a6553f100",
    "payload_b64":
      "pAF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQOBeB1odHRwczovL2
       1vcS1yZWxheS5leGFtcGxlLmNvbQQaZVVCgAUaZVPxAA",
    "signature_hex":
      "fa3315e9de061fd77d814394428ae61da3d7a21fdffb19802b0c575c
       578098e7cd4b6b75a1690deed4c2baae994bfc462e0d8a2006f3e897
       80f3435738294d7a",
    "signature_b64":
      "-jMV6d4GH9d9gUOUQormHaPXoh_f-xmAKwxXXFeAmOfNS2t1oWkN7t
       TCuq6ZS_xGLg2KIAbz6JeA80NXOClNeg",
    "private_key_hex":
      "c9afa9d845ba75166b5c215767b1d6934e50c3db36e89b127b8a622b
       120f6721",
    "public_key_x_hex":
      "60fed4ba255a9d31c961eb74c6356d68c049b8923b61fa6ce669622e
       60f29fb6",
    "public_key_y_hex":
      "7903fe1008b8bc99a41ae9e95628bc64f2f1b20c2d7e9f5177a3c294
       d4462299",
    "token":
      "ogEmEGNDQVQ.pAF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQOBeB
       1odHRwczovL21vcS1yZWxheS5leGFtcGxlLmNvbQQaZVVCgAUaZVPxA
       A.-jMV6d4GH9d9gUOUQormHaPXoh_f-xmAKwxXXFeAmOfNS2t1oWkN7
       tTCuq6ZS_xGLg2KIAbz6JeA80NXOClNeg",
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
      "Token with DPoP key binding (JWK thumbprint in cnf claim)",
    "dpop": {
      "cnf_jkt_hex":
        "a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1",
      "window_seconds": 60,
      "honor_jti": true
    },
    "payload_cbor_hex":
      "a401781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a6555428008a1035820a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6
       f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1190141a200183c0101",
    "token":
      "ogEjEGNDQVQ.pAF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQQaZV
       VCgAihA1ggoLHC0-T1prfI2eDxorPE1eb3qLnA0eLzpLXG1-j5oLEZA
       UGiABg8AQE.sN9kLIp64zIN9zDXoTLYC0xsJU_1FNF3kaO0CbdA_3M"
  },
  {
    "id": "dpop_no_jti",
    "description":
      "DPoP binding with longer window, JTI processing disabled",
    "dpop": {
      "cnf_jkt_hex":
        "3c82dfd6358ba804bd90879c34e743bbe13aeab7980664944f37a0ec0063fe95",
      "cnf_jkt_source": "SHA-256 of 'test-public-key-material'",
      "window_seconds": 300,
      "honor_jti": false
    },
    "payload_cbor_hex":
      "a401781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a6555428008a10358203c82dfd6358ba804bd90879c34e743bbe13a
       eab7980664944f37a0ec0063fe95190141a20019012c0100",
    "token":
      "ogEjEGNDQVQ.pAF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQQaZV
       VCgAihA1ggPILf1jWLqAS9kIecNOdDu-E66reYBmSUTzeg7ABj_pUZA
       UGiABkBLAEA.M4lF5pQdxav6eIWqDjbchDkijVYOM7xa3oJR2IwWt9g"
  },
  {
    "id": "dpop_es256_real_binding",
    "description":
      "ES256 token with real JWK thumbprint binding to the signing key",
    "algorithm": "ES256",
    "jwk_thumbprint_input":
      "{\"crv\":\"P-256\",\"kty\":\"EC\",\"x\":\"YP7UuiVanTHJYet0xjVtaMBJuJI7Yfps5mliLmDyn7Y\",\"y\":\"eQP-EAi4vJmkGunpVii8ZPLxsgwtfp9Rd6PClNRGIpk\"}",
    "dpop": {
      "cnf_jkt_hex":
        "0cebf1bc9880748a95588905b79843b42ba75cb174055e3e246bf87fe00b4a6d",
      "window_seconds": 120,
      "honor_jti": null
    },
    "public_key_x_hex":
      "60fed4ba255a9d31c961eb74c6356d68c049b8923b61fa6ce669622e60f29fb6",
    "public_key_y_hex":
      "7903fe1008b8bc99a41ae9e95628bc64f2f1b20c2d7e9f5177a3c294d4462299",
    "payload_cbor_hex":
      "a501781868747470733a2f2f617574682e6578616d706c652e636f6d
       0381781968747470733a2f2f72656c61792e6578616d706c652e636f6d
       041a6555428008a10358200cebf1bc9880748a95588905b79843b42ba7
       5cb174055e3e246bf87fe00b4a6d190141a1001878",
    "token":
      "ogEmEGNDQVQ.pQF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQOBeB
       lodHRwczovL3JlbGF5LmV4YW1wbGUuY29tBBplVUKACKEDWCAM6_G8m
       IB0ipVYiQW3mEO0K6dcsXQFXj4ka_h_4AtKbRkBQaEAGHg.5andoxOh
       WXQIKWR3EMHWT-WIMPBDMYQFc61nzlfZs8zmgzwcOARpmlaB3ZS5MbJ
       9iCYWykYAcIzJ81nMyyZoQw"
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
       041a6555428019014781838202068 24b6578616d706c652e636f6d4561
       6c696365820146766964656f2d",
    "token":
      "ogEjEGNDQVQ.owF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQQaZV
       VCgBkBR4GDggIGgktleGFtcGxlLmNvbUVhbGljZYIBRnZpZGVvLQ.oA
       PD24Wu_zHnDcuM6a-ePeGvRJjbCa6U7iswdsKzFDk",
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
       041a65554280190147818283030407818201 52636f6e666572656e6365
       2e6578616d706c65",
    "token":
      "ogEjEGNDQVQ.owF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQQaZV
       VCgBkBR4GCgwMEB4GCAVJjb25mZXJlbmNlLmV4YW1wbGU.pfUPZultm
       yCm1GF2PvPXAYXzvK6d1D-OFNBLG1AwjDg",
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
      "Multi-scope token: publish to specific namespace, subscribe to prefix, with revalidation",
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
       041a655542801901478282820206824c6c6976652e6578616d706c6548
       73747564696f2d61828204078182014c6c6976652e6578616d706c6519
       0148f95cb0",
    "token":
      "ogEjEGNDQVQ.pAF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQQaZV
       VCgBkBR4KCggIGgkxsaXZlLmV4YW1wbGVIc3R1ZGlvLWGCggQHgYIBT
       GxpdmUuZXhhbXBsZRkBSPlcsA.byEzQmxc28UXFyFekHtgOtaVmWyIPl
       -63xNMOF0Q_IU",
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
       041a65554280190147818189000102030405060708",
    "token":
      "ogEjEGNDQVQ.owF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQQaZV
       VCgBkBR4GBiQABAgMEBQYHCA.XlNItz7OGqnNEbaqZ_bQh6TL-wV6SDr
       8hXyOLmtQkj4",
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
       041a6555428019014781838104818202 4c2e6578616d706c652e636f6d
       8202462d617564696f",
    "token":
      "ogEjEGNDQVQ.owF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQQaZV
       VCgBkBR4GDgQSBggJMLmV4YW1wbGUuY29tggJGLWF1ZGlv.-eGYTPe_n
       1PeC0sgHdWCqgnKRHGYF-T89WTk269liBg",
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

## Token Validation

These vectors validate token processing: expected pass and fail scenarios.
All tokens use HMAC-SHA256 with the key specified in the Keys section unless
otherwise noted.

~~~ json
[
  {
    "id": "valid_basic",
    "description":
      "Valid token with correct issuer, audience, and time bounds",
    "token":
      "ogEjEGNDQVQ.pAF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQOBeBlodHRwczovL3JlbGF5LmV4YW1wbGUuY29tBBplVUKABRplU_EA.9SztgnG4xgw8U9zDFnqPIuPn6hLwuilSigQcfPsArSg",
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
      "ogEjEGNDQVQ.ogF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQQaX14QAA.lq8nGBiZm80yUwl1kH_Tv2prKu_nV20JvxVJW8ZGkho",
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
      "ogEjEGNDQVQ.owF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQQaZVaUAAUaZVVCgA.fPIUugY7_oSeHlheu83_8Yyljsk3iP2zGeWRUi7NtUs",
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
      "ogEjEGNDQVQ.owF4GGh0dHBzOi8vZXZpbC5leGFtcGxlLmNvbQOBeBlodHRwczovL3JlbGF5LmV4YW1wbGUuY29tBBplVUKA.Xo7FCr_MGSyVX0C9sueeapSfboIHkrkysurn2VjC9PU",
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
      "ogEjEGNDQVQ.owF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQOBeB9odHRwczovL290aGVyLXJlbGF5LmV4YW1wbGUuY29tBBplVUKA.b8KxAKxJglzhELMuc9bYmsikrx3F9Y3YdvpfHLbsyk0",
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
      "Token with corrupted signature (first byte flipped)",
    "original_token":
      "ogEjEGNDQVQ.owF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQOBeBlodHRwczovL3JlbGF5LmV4YW1wbGUuY29tBBplVUKA.W17GD7Gj-B0YtejX7fRwLlUmEkje-ME81oCc9oZaaYY",
    "token":
      "ogEjEGNDQVQ.owF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQOBeBlodHRwczovL3JlbGF5LmV4YW1wbGUuY29tBBplVUKA.pF7GD7Gj-B0YtejX7fRwLlUmEkje-ME81oCc9oZaaYY",
    "validation": {
      "expected_result": "error",
      "expected_error": "SignatureVerificationFailed",
      "key_hex":
        "000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f"
    }
  },
  {
    "id": "invalid_wrong_key",
    "description": "Token verified with incorrect key",
    "token":
      "ogEjEGNDQVQ.ogF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQQaZVVCgA.zmbxdkvbWtGtX0DExLC2nIxPDDmgAVImqk4rRSCCkCY",
    "validation": {
      "correct_key_hex":
        "000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f",
      "wrong_key_hex":
        "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "expected_result": "error",
      "expected_error": "SignatureVerificationFailed"
    }
  },
  {
    "id": "invalid_algorithm_mismatch",
    "description":
      "Token header says HMAC-SHA256 but verifier expects ES256",
    "token":
      "ogEjEGNDQVQ.ogF4GGh0dHBzOi8vYXV0aC5leGFtcGxlLmNvbQQaZVVCgA.zmbxdkvbWtGtX0DExLC2nIxPDDmgAVImqk4rRSCCkCY",
    "validation": {
      "token_algorithm_id": -4,
      "verifier_algorithm_id": -7,
      "expected_result": "error",
      "expected_error": "AlgorithmMismatch"
    }
  }
]
~~~

# Acknowledgments
{:numbered="false"}

The IETF moq workgroup
