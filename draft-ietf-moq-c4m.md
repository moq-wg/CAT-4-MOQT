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

  Composite: I-D.draft-lemmons-cose-composite-claims-01
  MoQTransport: I-D.draft-ietf-moq-transport-16
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
actions authorized by it MUST not be permitted to continue.

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
| Specification Document | RFCthis        | RFCthis           |
|------------------------|----------------|-------------------|

\[RFC Editor: Please replace RFCthis with the published RFC number for this
document.\]

--- back

# Acknowledgments
{:numbered="false"}

The IETF moq workgroup
