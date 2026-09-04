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
  MoQTransport: I-D.draft-ietf-moq-transport-18
  EDN: I-D.draft-ietf-cbor-edn-literals
  BASE64: RFC4648
  CAT:
    title: "CTA 5007-B Common Access Token"
    date: April 2025
    target: https://shop.cta.tech/products/cta-5007
  DPoP: RFC9449
  DPOP-PROOF:
    title: "Application-Agnostic Demonstrating Proof-of-Possession"
    author:
      name: "S. Nandakumar"
    date: December 2024
    target: https://datatracker.ietf.org/doc/draft-nandakumar-moq-generic-dpop-proof/
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
        |     (PUBLISH_NAMESPACE, SUBSCRIBE, PUBLISH, FETCH)|
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

|----------------------|-----|-------------------------------|
| Action               | Key | Reference                     |
|----------------------|-----|-------------------------------|
| CLIENT_SETUP         |  0  | {{MoQTransport}} Section 9.3  |
| SERVER_SETUP         |  1  | {{MoQTransport}} Section 9.3  |
| PUBLISH_NAMESPACE    |  2  | {{MoQTransport}} Section 9.20 |
| SUBSCRIBE_NAMESPACE  |  3  | {{MoQTransport}} Section 9.25 |
| SUBSCRIBE            |  4  | {{MoQTransport}} Section 9.9  |
| REQUEST_UPDATE       |  5  | {{MoQTransport}} Section 9.11 |
| PUBLISH              |  6  | {{MoQTransport}} Section 9.13 |
| FETCH                |  7  | {{MoQTransport}} Section 9.16 |
| TRACK_STATUS         |  8  | {{MoQTransport}} Section 9.19 |
|----------------------|-----|-------------------------------|

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

# DPoP Integration with CAT for MOQT

This section defines the use of CAT's Demonstrating Proof of Possession (DPoP)
claims {{DPoP}} to enhance security in MOQT environments. This approach
leverages the CAT token's "cnf" (confirmation) claim with JWK Thumbprint
binding and the "catdpop" (CAT DPoP Settings) claim to provide
proof-of-possession capabilities that prevent token theft and replay
attacks in MOQT systems.

## CAT DPoP Claims for MOQT

This proposal extends the CAT authorization model by binding tokens to
client cryptographic key pairs. To enable sender-constrained token usage,
the CAT tokens include DPoP-related claims as defined {{CAT}} Section 4.8,
ensuring that only the legitimate token holder can use the token for MOQT
operations.

### Confirmation (cnf) Claim with JWK Thumbprint

DPoP binding is accomplished by providing the "cnf" claim with the "jkt"
(JWK Thumbprint) confirmation method.

Below is an example showing jkt token binding.

~~~~
{
  / cnf / 8: {
    / jkt / 3: h'0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef'  / 32-byte SHA-256 JWK thumbprint (hex-encoded) /
  },
  / moqt / TBD_MOQT: [
    [
      [/PUBLISH_NAMESPACE/ 2, /SUBSCRIBE_NAMESPACE/ 3, /PUBLISH/ 6, /FETCH/ 7],
      ['cdn','example','com',nil],
      [ /prefix/ 1, '/sports/']
    ]
  ],
  / catdpop /
  321: {
    0: 300,  / 5-minute window /
    1: 1     / Honor jti for replay protection /
  },
  / exp /
  4: 1750000000
}
~~~~

Implementation Requirements:

- Relay Validation: MOQT relays MUST verify that DPoP proofs are signed with
  the private key corresponding to the "jkt" value
- Proof Binding: Relays MUST reject requests where DPoP proof validation or
  key binding fails
- Processing Semantics: Relays MUST process DPoP proofs as Protected Resource
  Access requests per {{DPoP}} Section 7


### DPoP Extension with Application-Agnostic Proof Framework

This section defines the use of DPoP with an application-agnostic proof
framework as specified in {{DPOP-PROOF}}, which
extends the traditional HTTP-centric DPoP model to support arbitrary
protocols including MOQT. This approach replaces HTTP-specific claims
with a flexible authorization context structure that can accommodate
protocol-specific command representations.

The DPoP proof JWT follows the structure defined in Section 4 of
{{DPOP-PROOF}} with the following required claims:

JWT Header:

- "typ": "dpop-proof+jwt"
- "alg": Asymmetric signature algorithm identifier
- "jwk": Public key for verification

JWT Payload:

- "jti": Unique identifier for the JWT
- "iat": Issued-at time
- "actx": Authorization Context object

For MOQT operations, the Authorization Context ("actx") object contains:

- "type": "moqt" (registered identifier for MOQT protocol)
- "action": MOQT action identifier
- "tns": Track namespace (required)
- "tn": Track name (required)
- "resource": MOQT resource identifier (optional)

When the optional "resource" parameter is included, it MUST be consistent with the
"tns" and "tn" parameters. The resource URI should follow the format
`moqt://<relay-endpoint>?tns=<namespace>&tn=<track>` where the tns and tn query
parameters match the respective "tns" and "tn" fields in the Authorization Context.

Example DPoP proof for MOQT PUBLISH_NAMESPACE operation:

~~~~~~~~~~~~~~~
{
  "typ": "dpop-proof+jwt",
  "alg": "ES256",
  "jwk": { ... }
}
.
{
  "jti": "unique-request-id",
  "iat": 1705123456,
  "actx": {
    "type": "moqt",
    "action": "PUB_NS",
    "tns": "sports",
    "tn": "live-feed"
  }
}
~~~~~~~~~~~~~~~

MOQT action mapping for Authorization Context:

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

Relays supporting this application-agnostic DPoP framework MUST:

- Validate DPoP proofs according to {{DPOP-PROOF}}
- Verify that the "actx.type" is "moqt" for MOQT operations
- Validate that the "actx.action" matches the requested MOQT action
- Verify that the "actx.tns" corresponds to the target track namespace
- Verify that the "actx.tn" corresponds to the target track name
- If present, verify the "actx.resource" is consistent with "tns" and "tn"
- Reject requests where Authorization Context validation fails

### MOQT Resource URI Construction

The Authorization Context "resource" field should specify track namespace (tns) and track name (tn) parameters for MOQT resources:

- Connection setup: `moqt://<relay-endpoint>`
- Namespace operations: `moqt://<relay-endpoint>?tns=<namespace>`
- Track operations: `moqt://<relay-endpoint>?tns=<namespace>&tn=<track>`

## DPoP Proof Process and Token Binding Flow

The following process illustrates how DPoP proof provision results in CAT
token binding and subsequent MOQT relay validation:

### Phase 1: Token Acquisition with DPoP Binding

~~~~
┌──────────────┐                ┌─────────────────────┐                ┌──────┐
│MOQT Client   │                │Authorization Server │                │MOQT  │
│              │                │                     │                │Relay │
└──────┬───────┘                └──────────┬──────────┘                └──────┘
       │                                   │                                │
       │ (1) Generate Key Pair             │                                │
       │     EC P-256/RSA                  │                                │
       │     private_key, public_key       │                                │
       │                                   │                                │
       │ (2) Authentication Request        │                                │
       │     + User Credentials            │                                │
       │     + Public Key (JWK format)     │                                │
       ├──────────────────────────────────►│                                │
       │                                   │                                │
       │                                   │ (3) User Authentication        │
       │                                   │     & Authorization            │
       │                                   │                                │
       │                                   │ (4) Generate CAT Token:        │
       │                                   │     • "cnf" claim with         │
       │                                   │       "jkt": SHA256(public_key)│
       │                                   │     • "catdpop" processing     │
       │                                   │       settings                 │
       │                                   │     • "moqt" action scope      │
       │                                   │     • Sign with shared secret  │
       │                                   │                                │
       │ (5) CAT Token Response            │                                │
       │     + Bound CAT Token             │                                │
       │     + Relay Endpoint URL          │                                │
       |◄──────────────────────────────────┤                                │
       │                                   │                                │
~~~~

Steps 1-5 Detail:

1. Client Key Generation: The MOQT client generates an asymmetric key pair
(typically EC P-256) for DPoP operations
2. Authentication with Public Key: Client authenticates with the authorization
   server, providing user credentials and the public key
3. User Authentication: Authorization server validates user identity and
permissions
1. CAT Token Generation: Server creates a CAT token containing:
   - "cnf" claim: JWK Thumbprint ("jkt") of the client's public key
     (32-byte SHA-256 hash)
   - "catdpop" claim: DPoP processing settings (window, jti handling,
     critical settings)
   - "moqt" claim: Authorized MOQT actions and scope restrictions
2. Token Delivery: Server provides the bound CAT token and relay endpoint
   information to the client

### Phase 2: MOQT Operations with DPoP Proof Validation

~~~~
┌──────────────┐                ┌─────────────────────┐                ┌───────┐
│MOQT Client   │                │Authorization Server │                │MOQT   │
│              │                │                     │                │Relay  │
└──────┬───────┘                └──────────┬──────────┘                └──────┬┘
       │                                   │                                  │
       │                                   │                                  │
       │ (6) For each MOQT action:         │                                  │
       │     Create fresh DPoP proof JWT   │                                  │
       │     • Header: typ="dpop-proof+jwt"│                                  │
       │     •         alg, jwk            │                                  │
       │     • Claims: jti, iat, actx      │                                  │
       │     • Sign with private_key       │                                  │
       │                                   │                                  │
       │ (7) MOQT Request                  │                                  │
       │     + CAT Token                   │                                  │
       │     + Fresh DPoP Proof            │                                  │
       │     (CLIENT_SETUP, PUBLISH_NAMESPACE,│                                │
       │      SUBSCRIBE, PUBLISH, FETCH)   │                                  │
       ├─────────────────────────────────────────────────────────────────────►│
       │                                   │                                  │
       │                                   │                               (8)│
       │                                   │                  CAT Validation: │
       │                                   │                 • Verify token   │
       │                                   │                   signature      │
       │                                   │                 • Validate claims│
       │                                   │                   including exp, |
       |                                   |                   scope          │
       │                                   │                                  │
       │                                   │                               (9)│
       │                                   │                 DPoP Validation: │
       │                                   │                  • Extract "jkt" │
       │                                   │                    from token    │
       │                                   │                  • Verify DPoP   │
       │                                   │                    JWT signature │
       │                                   │                  • Validate key  │
       │                                   │                    binding       │
       │                                   │                  • Check         │
       │                                   │                    freshness     │
       │                                   │                                  │
       │                                   │                              (10)│
       │                                   │              Action Authorization│
       │                                   │                  • Match action  │
       │                                   │                    to token scope│
       │                                   │                  • Check ns/track│
       │                                   │                    permissions   │
       │                                   │                                  │
       │ (11) Response                     │                                  │
       │      Success/Error                │                                  │
       ◄─────────────────────────────────────────────────────────────────────┤
       │                                   │                                  │
~~~~

Steps 6-11 Detail:

6. DPoP Proof Creation: For each MOQT action, the client creates a fresh
  DPoP proof JWT with:
   - Header: `typ: "dpop-proof+jwt"`, `alg`, `jwk` (public key)
   - Claims: `jti` (unique ID), `iat` (timestamp), `actx`
             (Authorization Context with type, action, tns, tn)

1. MOQT Request: Client sends MOQT action with both CAT token and fresh DPoP
  proof

2. CAT Token Validation: Relay validates:
   - Token signature using shared secret with authorization server
   - Token expiration time
   - "moqt" claim scope for requested action

3. DPoP Proof Validation: Relay performs:

   - Extract "jkt" (JWK Thumbprint) from CAT token's "cnf" claim
   - Verify DPoP JWT signature using embedded public key
   - Confirm that SHA-256 hash of DPoP public key matches "jkt" value
   - Check proof freshness within "catdpop" window settings
   - Process replay protection based on "jti" settings
   - Validate Authorization Context ("actx") according to {{DPOP-PROOF}}
   - Verify "actx.type" is "moqt"
   - Validate "actx.action" matches the requested MOQT action
   - Verify "actx.tns" and "actx.tn" correspond to target resources

4. Action Authorization: Relay validates the specific MOQT action against
   token scope and namespace/track permissions

5.  Response: Relay responds with success or appropriate error information

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

TODO Add security considerations for DPoP Claims


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

## MOQT Auth Token Type Registry

This document registers the following entry in the "MOQT Auth Token Type"
registry established by {{MoQTransport}}:

|-------------|-------------------|---------------------------|
| Token Type  | Token Name        | Specification             |
|-------------|-------------------|---------------------------|
| 0x01        | CAT               | RFCXXXX                   |
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
- Process DPoP claims (if present) according to Section 3 of this document

If the token fails validation, the relay MUST reject the connection or
action with an appropriate error.

--- back

#  Appendix A: Test Vectors

This appendix provides test vectors in JSON format for cross-implementation
validation of CAT tokens for MOQT. Tokens use COSE_Mac0 (CBOR tag 17) for
HMAC-SHA256 or COSE_Sign1 (CBOR tag 18) for ES256. Token strings are the
base64url encoding of the full COSE structure.

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
      "a501781868747470733a2f2f617574682e6578616d706c652e636f6d
       0381781968747470733a2f2f72656c61792e6578616d706c652e636f
       6d041a65554280051a6553f100074e746573742d746f6b656e2d3030
       31"
  },
  {
    "id": "cbor_cat_version_uri",
    "description": "CAT version (uint 1) and URI match rule",
    "claims": {
      "catv": 1,
      "catu": {"host": "example.com"}
    },
    "payload_cbor_hex":
      "a219013601190138a101a1006b6578616d706c652e636f6d"
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
      "a119013784d83444c0a80164d834a108410a19fc008219fc0019fd00"
  },
  {
    "id": "cbor_geographic_claims",
    "description":
      "Geographic claims: coordinates, geohash, ISO 3166, altitude",
    "claims": {
      "geohash": "9q8yyk",
      "catgeoiso3166": ["US", "CA"],
      "catgeocoord": {"lat": 37.7749, "lon": -122.4194,
                      "accuracy": 100.0},
      "catgeoalt": 10
    },
    "payload_cbor_hex":
      "a419011a6639713879796b19013c82625553624341
       19013d8183fb4042e32fec56d5d0fbc05e9ad77318fc5018
       6419013e820a05"
  },
  {
    "id": "cbor_uri_match_rules",
    "description":
      "URI match rules: host exact, path prefix, extension exact",
    "claims": {
      "catu": [
        {"component": "host", "match": "exact",
         "value": "example.com"},
        {"component": "path", "match": "prefix",
         "value": "/vod/"},
        {"component": "extension", "match": "exact",
         "value": "m3u8"}
      ]
    },
    "payload_cbor_hex":
      "a1190138a301a1006b6578616d706c652e636f6d03a101652f766f64
       2f08a100646d337538"
  },
  {
    "id": "cbor_alpn",
    "description": "ALPN protocol identifiers",
    "claims": {
      "catalpn": ["moq-00", "h3"]
    },
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
    "algorithm": "HMAC-SHA256",
    "algorithm_id": 5,
    "claims": {
      "iss": "https://auth.example.com",
      "aud": ["https://relay.example.com"],
      "exp": 1700086400
    },
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
      "d18448a201051063434154a0583fa301781868747470733a2f2f61757
       4682e6578616d706c652e636f6d0381781968747470733a2f2f72656c
       61792e6578616d706c652e636f6d041a65554280582016ca16a2d4e24
       76528ae858d91b00f257a507fe57284bfb13d7d6ca1f065e230",
    "cose_b64":
      "0YRIogEFEGNDQVSgWD-jAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tA4F4GWh0dHBzOi8vcmVsYXkuZXhhbXBsZS5jb20EGmVVQoBYIB
       bKFqLU4kdlKK6FjZGwDyV6UH_lcoS_sT19bKHwZeIw",
    "valid": true
  },
  {
    "id": "token_hmac_full",
    "description":
      "COSE_Mac0 token with core + CAT + informational claims,
       HMAC-SHA256",
    "algorithm": "HMAC-SHA256",
    "algorithm_id": 5,
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
      "catv": 1,
      "catnip": [{"type": "ip_address", "value": "203.0.113.50"}],
      "catu": {"path": "/live/"}
    },
    "header_cbor_hex": "a201051063434154",
    "payload_cbor_hex":
      "aa01781a68747470733a2f2f6973737565722e6d6f712e6578616d70
       6c650276757365723a616c696365406578616d706c652e636f6d0382
       781a68747470733a2f2f72656c6179312e6578616d706c652e636f6d
       781a68747470733a2f2f72656c6179322e6578616d706c652e636f6d
       041a65554280051a6553f100061a6553f100074a766563746f722d30
       30321901360119013781d83444cb007132190138a103a101662f6c697
       6652f",
    "tag_hex":
      "29c1a6612ce6d49273fd8c0116b9decbc287662d5ba5d67defafaf6f
       f0c11278",
    "key_hex":
      "000102030405060708090a0b0c0d0e0f101112131415161718191a1b
       1c1d1e1f",
    "cose_hex":
      "d18448a201051063434154a058abaa01781a68747470733a2f2f69737
       37565722e6d6f712e6578616d706c650276757365723a616c69636540
       6578616d706c652e636f6d0382781a68747470733a2f2f72656c61793
       12e6578616d706c652e636f6d781a68747470733a2f2f72656c617932
       2e6578616d706c652e636f6d041a65554280051a6553f100061a65530
       f100074a766563746f722d3030321901360119013781d83444cb007132
       190138a103a101662f6c6976652f582029c1a6612ce6d49273fd8c011
       6b9decbc287662d5ba5d67defafaf6ff0c11278",
    "cose_b64":
      "0YRIogEFEGNDQVSgWKuqAXgaaHR0cHM6Ly9pc3N1ZXIubW9xLmV4
       YW1wbGUCdnVzZXI6YWxpY2VAZXhhbXBsZS5jb20DgngaaHR0cHM6Ly
       9yZWxheTEuZXhhbXBsZS5jb214Gmh0dHBzOi8vcmVsYXkyLmV4YW1
       wbGUuY29tBBplVUKABRplU_EABhplU_EAB0p2ZWN0b3ItMDAyGQE2
       ARkBN4HYNETLAHEyGQE4oQOhAWYvbGl2ZS9YICnBpmEs5tSSc_2MAR
       a53svCh2YtW6XWfe-vr2_wwRJ4",
    "valid": true
  },
  {
    "id": "token_es256",
    "description":
      "COSE_Sign1 token signed with ES256
       (P-256 ECDSA, deterministic RFC 6979)",
    "algorithm": "ES256",
    "algorithm_id": -7,
    "claims": {
      "iss": "https://auth.example.com",
      "aud": ["https://moq-relay.example.com"],
      "exp": 1700086400,
      "nbf": 1700000000
    },
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
      "d28448a201261063434154a05849a401781868747470733a2f2f61757
       4682e6578616d706c652e636f6d0381781d68747470733a2f2f6d6f71
       2d72656c61792e6578616d706c652e636f6d041a65554280051a6553f
       1005840ed8cf586f32ba410f2b3097642eb32255c8dc555e9e058c4808
       4809df557887bc5ee4974a9ef5c4557c1557de799628b634cafaa1b74
       c936968e0c9c3a48f222",
    "cose_b64":
      "0oRIogEmEGNDQVSgWEmkAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tA4F4HWh0dHBzOi8vbW9xLXJlbGF5LmV4YW1wbGUuY29tBBplVU
       KABRplU_EAWEDtjPWG8yukEPKzCXZC6zIlXI3FVengWMSAhICd9VeI
       e8XuSXSp71xFV8FVfeeZYotjTK-qG3TJNpaODJw6SPIi",
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
        "a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6
         d7e8f9a0b1",
      "window_seconds": 60,
      "honor_jti": true
    },
    "payload_cbor_hex":
      "a401781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a6555428008a11901435820a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4
       d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1190141a200183c0101",
    "cose_hex":
      "d18448a201051063434154a05852a401781868747470733a2f2f61757
       4682e6578616d706c652e636f6d041a6555428008a11901435820a0b1
       c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8
       f9a0b1190141a200183c01015820353668 6cfff58bbafc41f95a4ae21d
       e501a16cabe0a4b15086940bfb8fea92d9",
    "cose_b64":
      "0YRIogEFEGNDQVSgWFKkAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tBBplVUKACKEZAUNYIKCxwtPk9aa3yNng8aKzxNXm96i5wNHi86
       S1xtfo-aCxGQFBogAYPAEBWCA1Zrhs__WLuvxB-VpK4h3lAaFsq-Ck
       sVCGlAv7j-qS2Q"
  },
  {
    "id": "dpop_no_jti",
    "description":
      "DPoP binding with longer window, JTI processing disabled",
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
       041a6555428008a119014358203c82dfd6358ba804bd90879c34e743bb
       e13aeab7980664944f37a0ec0063fe95190141a20019012c0100",
    "cose_hex":
      "d18448a201051063434154a05853a401781868747470733a2f2f61757
       4682e6578616d706c652e636f6d041a6555428008a119014358203c82
       dfd6358ba804bd90879c34e743bbe13aeab7980664944f37a0ec0063f
       e95190141a20019012c01005820ecb7a5b5667b95f20c673417718eac
       9f5d997fd427485942a61048cdd8c50e0c",
    "cose_b64":
      "0YRIogEFEGNDQVSgWFOkAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tBBplVUKACKEZAUNYIDyC39Y1i6gEvZCHnDTnQ7vhOuq3mAZklE
       83oOwAY_6VGQFBogAZASwBAFgg7LeltWZ7lfIMZzQXcY6sn12Zf9Qn
       SFlCphBIzdjFDgw"
  },
  {
    "id": "dpop_es256_real_binding",
    "description":
      "ES256 token with real JWK thumbprint binding to the
       signing key",
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
       6d041a6555428008a119014358200cebf1bc9880748a95588905b7984
       3b42ba75cb174055e3e246bf87fe00b4a6d190141a1001878",
    "cose_hex":
      "d28448a201261063434154a0586da501781868747470733a2f2f61757
       4682e6578616d706c652e636f6d0381781968747470733a2f2f72656c
       61792e6578616d706c652e636f6d041a6555428008a1190143582 00ce
       bf1bc9880748a95588905b79843b42ba75cb174055e3e246bf87fe00b
       4a6d190141a10018785840cdb69dc3db876b1d05a8f808e5286a8610
       26fc8b58b9a9aa97f23bdf0253ff56357f9129ee9b09667701357c8d
       700721b3029928d4fe573d56a88744dce04f82",
    "cose_b64":
      "0oRIogEmEGNDQVSgWG2lAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tA4F4GWh0dHBzOi8vcmVsYXkuZXhhbXBsZS5jb20EGmVVQoAIoR
       kBQ1ggDOvxvJiAdIqVWIkFt5hDtCunXLF0BV4-JGv4f-ALSm0ZAUGh
       ABh4WEDNtp3D24drHQWo-AjlKGqGECb8i1i5qaqX8jvfAlP_VjV_kS
       numwlmdwE1fI1wByGzApko1P5XPVaoh0Tc4E-C"
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
       041a655542801901478183820206824b6578616d706c652e636f6d4561
       6c696365820146766964656f2d",
    "cose_hex":
      "d18448a201051063434154a05846a301781868747470733a2f2f61757
       4682e6578616d706c652e636f6d041a655542801901478183820206824
       b6578616d706c652e636f6d45616c696365820146766964656f2d5820
       e6ef1eb0edbd76e780a9e46f7077edcab091b55d7991aa9c1ba5d2b17
       711e59a",
    "cose_b64":
      "0YRIogEFEGNDQVSgWEajAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tBBplVUKAGQFHgYOCAgaCS2V4YW1wbGUuY29tRWFsaWNlggFGdm
       lkZW8tWCDm7x6w7b1254Cp5G9wd-3KsJG1XXmRqpwbpdKxdxHlmg",
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
       041a6555428019014781828303040781820152636f6e666572656e6365
       2e6578616d706c65",
    "cose_hex":
      "d18448a201051063434154a05841a301781868747470733a2f2f61757
       4682e6578616d706c652e636f6d041a65554280190147818283030407
       81820152636f6e666572656e63652e6578616d706c655820a517009de
       ce13f143b06cd1f9e5fd861a29338174b77ed7c0d0478f1bc55fea2",
    "cose_b64":
      "0YRIogEFEGNDQVSgWEGjAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tBBplVUKAGQFHgYKDAwQHgYIBUmNvbmZlcmVuY2UuZXhhbXBsZV
       ggpRcAnezhPxQ7Bs0fnl_YYaKTOBdLd-18DQR48bxV_qI",
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
       014819012c",
    "cose_hex":
      "d18448a201051063434154a0585ba401781868747470733a2f2f61757
       4682e6578616d706c652e636f6d041a65554280190147828282020682
       4c6c6976652e6578616d706c654873747564696f2d6182820407818201
       4c6c6976652e6578616d706c6519014819012c58204d0c6f8c520acc96
       47637833b1773a2bad38595a261f3cdddfcf6edecf312b4e",
    "cose_b64":
      "0YRIogEFEGNDQVSgWFukAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tBBplVUKAGQFHgoKCAgaCTGxpdmUuZXhhbXBsZUhzdHVkaW8tYY
       KCBAeBggFMbGl2ZS5leGFtcGxlGQFIGQEsWCBNDG-MUgrMlkdjeDOx
       dzorrThZWiYfPN3fz27ezzErTg",
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
    "cose_hex":
      "d18448a201051063434154a05831a301781868747470733a2f2f61757
       4682e6578616d706c652e636f6d041a655542801901478181890001020
       3040506070858202bacff08b2be7762aa9ec246bbf0c2d8e12b05e14a
       7bf5b73f85c89f7d6d61fd",
    "cose_b64":
      "0YRIogEFEGNDQVSgWDGjAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tBBplVUKAGQFHgYGJAAECAwQFBgcIWCArrP8Isr53Yqqewka78ML
       Y4SsF4Up79bc_hciffW1h_Q",
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
       041a65554280190147818381048182024c2e6578616d706c652e636f6d
       8202462d617564696f",
    "cose_hex":
      "d18448a201051063434154a05842a301781868747470733a2f2f61757
       4682e6578616d706c652e636f6d04 1a6555428019014781838104818202
       4c2e6578616d706c652e636f6d8202462d617564696f58206fa814835
       0e44c3bae4c92753fd292c358776beacb6866a4c6738d80a51b9cd8",
    "cose_b64":
      "0YRIogEFEGNDQVSgWEKjAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUu
       Y29tBBplVUKAGQFHgYOBBIGCAkwuZXhhbXBsZS5jb22CAkYtYXVkaW
       9YIG-oFINQ5Ew7rkySdT_SksNYd2vqy2hmpMZzjYClG5zY",
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
token (iss, exp) with embedded composite claims (OR key 324, NOR key 325,
AND key 326). Tokens use COSE_Mac0 with HMAC-SHA256.

~~~ json
[
  {
    "id": "composite_or_simple",
    "description":
      "OR composite: at least one of two alternative claim sets
       must be acceptable",
    "composite": {
      "operator": "OR",
      "claim_key": 324,
      "claim_sets": [
        {"iss": "https://auth.example.com",
         "exp": 1700086400, "catv": 1},
        {"iss": "https://auth-backup.example.com",
         "exp": 1700090000, "catv": 1}
      ]
    },
    "payload_cbor_hex":
      "a301781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a6555428019014482a301781868747470733a2f2f617574682e65
       78616d706c652e636f6d041a6555428019013601a301781f68747470
       733a2f2f617574682d6261636b75702e6578616d706c652e636f6d04
       1a6555509019013601",
    "cose_hex":
      "d18448a201051063434154a05879a301781868747470733a2f2f6175
       74682e6578616d706c652e636f6d041a6555428019014482a3017818
       68747470733a2f2f617574682e6578616d706c652e636f6d041a6555
       428019013601a301781f68747470733a2f2f617574682d6261636b75
       702e6578616d706c652e636f6d041a655550901901360158203f11f8
       0896554afc5246d8711ca88317228a84d44fcb212563bc24812623d7
       49",
    "cose_b64":
      "0YRIogEFEGNDQVSgWHmjAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       BBplVUKAGQFEgqMBeBhodHRwczovL2F1dGguZXhhbXBsZS5jb20EGmVV
       QoAZATYBowF4H2h0dHBzOi8vYXV0aC1iYWNrdXAuZXhhbXBsZS5jb20E
       GmVVUJAZATYBWCA_EfgIllVK_FJG2HEcqIMXIoqE1E_LISVjvCSBJiPX
       SQ"
  },
  {
    "id": "composite_and",
    "description":
      "AND composite: both claim sets must be acceptable",
    "composite": {
      "operator": "AND",
      "claim_key": 326,
      "claim_sets": [
        {"exp": 1700086400, "catv": 1},
        {"exp": 1700086400,
         "catnip": [{"type": "ip_address", "value": "10.0.0.0"}]}
      ]
    },
    "payload_cbor_hex":
      "a301781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a6555428019014682a2041a6555428019013601a2041a65554280
       19013781d834440a000000",
    "cose_hex":
      "d18448a201051063434154a05843a301781868747470733a2f2f6175
       74682e6578616d706c652e636f6d041a6555428019014682a2041a65
       55428019013601a2041a6555428019013781d834440a0000005820b2
       41ea8429f9bdf073dea3e4a738f0dee7116d78e509e4f6fec9c8a13b
       8896de",
    "cose_b64":
      "0YRIogEFEGNDQVSgWEOjAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       BBplVUKAGQFGgqIEGmVVQoAZATYBogQaZVVCgBkBN4HYNEQKAAAAWCCy
       QeqEKfm98HPeo-SnOPDe5xFteOUJ5Pb-ycihO4iW3g"
  },
  {
    "id": "composite_nor",
    "description":
      "NOR composite: none of the listed claim sets can be
       acceptable",
    "composite": {
      "operator": "NOR",
      "claim_key": 325,
      "claim_sets": [
        {"iss": "https://revoked.example.com", "exp": 1700086400}
      ]
    },
    "payload_cbor_hex":
      "a301781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a6555428019014581a201781b68747470733a2f2f7265766f6b65
       642e6578616d706c652e636f6d041a65554280",
    "cose_hex":
      "d18448a201051063434154a0584ba301781868747470733a2f2f6175
       74682e6578616d706c652e636f6d041a6555428019014581a201781b
       68747470733a2f2f7265766f6b65642e6578616d706c652e636f6d04
       1a6555428058202cf030a29072032aa0775e71c2c2703a668fa2dbf1
       152d74258f0f42c4640d28",
    "cose_b64":
      "0YRIogEFEGNDQVSgWEujAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       BBplVUKAGQFFgaIBeBtodHRwczovL3Jldm9rZWQuZXhhbXBsZS5jb20E
       GmVVQoBYICzwMKKQcgMqoHdeccLCcDpmj6Lb8RUtdCWPD0LEZA0o"
  },
  {
    "id": "composite_nested",
    "description":
      "Nested composite: OR containing a standalone claim set
       and a nested AND",
    "composite": {
      "operator": "OR",
      "claim_key": 324,
      "claim_sets": [
        {"iss": "https://primary.example.com", "exp": 1700086400},
        {"nested_and": {
           "claim_key": 326,
           "claim_sets": [
             {"iss": "https://secondary.example.com",
              "exp": 1700086400},
             {"exp": 1700086400, "catv": 1}
           ]
         }
        }
      ]
    },
    "payload_cbor_hex":
      "a301781868747470733a2f2f617574682e6578616d706c652e636f6d
       041a6555428019014482a201781b68747470733a2f2f7072696d6172
       792e6578616d706c652e636f6d041a65554280a119014682a201781d
       68747470733a2f2f7365636f6e646172792e6578616d706c652e636f
       6d041a65554280a2041a6555428019013601",
    "cose_hex":
      "d18448a201051063434154a05882a301781868747470733a2f2f6175
       74682e6578616d706c652e636f6d041a6555428019014482a201781b
       68747470733a2f2f7072696d6172792e6578616d706c652e636f6d04
       1a65554280a119014682a201781d68747470733a2f2f7365636f6e64
       6172792e6578616d706c652e636f6d041a65554280a2041a65554280
       1901360158201577ec93556a6472e1e68846d3927211a531ddd337b1
       1e4c2d5e4e24f16776aa",
    "cose_b64":
      "0YRIogEFEGNDQVSgWIKjAXgYaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29t
       BBplVUKAGQFEgqIBeBtodHRwczovL3ByaW1hcnkuZXhhbXBsZS5jb20E
       GmVVQoChGQFGgqIBeB1odHRwczovL3NlY29uZGFyeS5leGFtcGxlLmNv
       bQQaZVVCgKIEGmVVQoAZATYBWCAVd-yTVWpkcuHmiEbTknIRpTHd0zex
       HkwtXk4k8Wd2qg"
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
