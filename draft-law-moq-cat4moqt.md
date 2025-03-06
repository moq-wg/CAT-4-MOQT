---
title: "Authentication scheme for MOQT using Common Access Tokens"
abbrev: "CAT-4-MOQT"
category: info

docname: draft-law-moq-cat4moqt-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: ""
workgroup: "Media Over QUIC"
keyword:
 - media over quic
 - authentication
 - common access token
 - CAT
venue:
  group: "Media Over QUIC"
  type: ""
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "wilaw/CAT-4-MOQT"
  latest: "https://wilaw.github.io/CAT-4-MOQT/draft-law-moq-cat4moqt.html"


author:

  -
    ins: W. Law
    name: "Will Law"
    organization: Akamai
    email: wilaw@akamai.com

  -
    ins: S. Nandakumar
    name: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com

normative:
  MoQTransport: I-D.draft-ietf-moq-transport-10
  BASE64: RFC4648
  CAT:
    title: "CTA 5007-A Common Access Token"
    date: December 2024
    target: https://shop.cta.tech/products/cta-5007

informative:


--- abstract

A token-based authentication scheme for use with Media Over QUIC Transport.


--- middle

# Introduction

This draft introduces a token-based authentication scheme for use with MOQT {{MoQTransport}}.
The scheme protects access to the relay during session establishment and also contrains the
actions which the client may take once connected.

This draft defines 1 of this specification.

## Overview of the authentication workflow

* An end-user logs-in to a distribution service. The service authenticates the user (via
  username/password, OAuth, 2FA or another method). The methods involved in this authentication step
  lie outside the scope of this draft.
* Based upon the identity and permissions granted to that end-user, the service generates a token. A
  token is a data structure that has been serialized into a byte array. The token encodes information
  such as the user's ID, constraints on how and when they can access the MOQT distribution network and
  contraints on the actions they can take once connected. The token may be signed to make it
  tamper-resistent.
* The token is given in the clear to the end-user, along with a URL to connect to the edge relay of a MOQT
  distribution network.
* The end-user client application provides the token to the MOQT distribution relay when it connects. This
  connection may be established over WebTransport or raw QUIC.
* The relay decrypts the token upon receipt and validates the signature, using secrets previously shared
  between the content distributor and the distribution network. Based upon claims conveyed in the token,
  relay will accept or reject the conneciton.
* If the relay accepts the connection, then the client will take a series of MOQT actions: ANNOUNCE,
  SUBSCRIBE_ANNOUNCES, SUBSCRIBE or FETCH. For each of these, it will supply the token it received using
  the AUTHENTICATION parameter.
* As an alternative to this workflow, the distribution service may vend multiple tokens to the client. The
  client may use one of those tokens to establish the initial conneciton and others to authenticate its actions.

# Token format

This draft uses a single token format, namely the Common Access Token (CAT) {{CAT}}. The token is supplied
as a byte array. When it must be cast to a string for inclusion in a URL, it is Base64 encoded {{BASE64}}.





# Authenticating the connection

The connection to a MOQT distribution realy can take place over a Webtransport of native QUIC connection. In
both cases, the token is transferred as a query parameter or else embedded in the URI PATH.

## Appending a token as a query parameter

The query parameter name SHOULD be "CAT" (case-sensitive) and the query parameter value SHOULD be the Base64 encoded {{BASE64}}
token. If more than one token is transferred, then the sequential query parameter names "CAT1", "CAT2" .. "CATN"
SHOULD be used.

## Embedding a token in a PATH

The token SHOULD span only a single PATH component and the component SHOULD be prefixed with the string "CAT-".
If more than one token is transferred, then they SHOULD occupy different components and SHOULD carry sequential
prefixes of "CAT1", "CAT2" .. "CATN".

## Usage with WebTransport
With a WebTransport connection, the token can be transferred as a query parameter or as part of the PATH.

Example of a single token in a query arg:
<blockquote>
https://example.com/service?CAT=oRkBDqMAoQBlaHR0cHMDoQFoL2NvbnRlbnQIoQBlLm0zdTg=
</blockquote>

Example of multiple tokens in query args:
<blockquote>
https://example.com/service?CAT1=oRkBDqMAoQBlaHR0cHMDoQFoL2NvbnRlbnQIoQBlLm0zdTg=&CAT2=IHNramRoZmtjc2pkaGYgc2pkaCBha
CBzIGFzS0pEIDthbGtqIA==
</blockquote>

Example of a single token in the PATH
<blockquote>
https://example.com/service/CAT-oRkBDqMAoQBlaHR0cHMDoQFoL2NvbnRlbnQIoQBlLm0zdTg=/
</blockquote>

Example of multiple tokens in the PATH:
<blockquote>
https://example.com/service/CAT1-oRkBDqMAoQBlaHR0cHMDoQFoL2NvbnRlbnQIoQBlLm0zdTg=/CAT2-IHNramRoZmtjc2pkaGYgc2pkaCBha
CBzIGFzS0pEIDthbGtqIA==/
</blockquote>

### Native QUIC example
With a native QUIC connection, the query components and PATH are transmitted via the "PATH" parameter in the CLIENT_SETUP message.

Example of a single token in a query arg:
<blockquote>
moqt://203.0.113.0:4443
PATH parameter in the CLIENT_SETUP message = "service?CAT=oRkBDqMAoQBlaHR0cHMDoQFoL2NvbnRlbnQIoQBlLm0zdTg="
</blockquote>

Example of multiple tokens in query args:
<blockquote>
moqt://203.0.113.0:4443
PATH parameter in the CLIENT_SETUP message = "service?CAT1=oRkBDqMAoQBlaHR0cHMDoQFoL2NvbnRlbnQIoQBlLm0zdTg=&CAT2=IHNramRoZmtjc2pka
GYgc2pkaCBhaCBzIGFzS0pEIDthbGtqIA=="
</blockquote>

Example of a single token in the PATH
<blockquote>
moqt://203.0.113.0:4443
PATH parameter in the CLIENT_SETUP message = "service/CAT-oRkBDqMAoQBlaHR0cHMDoQFoL2NvbnRlbnQIoQBlLm0zdTg=/"
</blockquote>

Example of multiple tokens in the PATH:
<blockquote>
moqt://203.0.113.0:4443
PATH parameter in the CLIENT_SETUP message = "service/CAT1-oRkBDqMAoQBlaHR0cHMDoQFoL2NvbnRlbnQIoQBlLm0zdTg=/CAT2-IHNramRoZmtjc2pka
GYgc2pkaCBhaCBzIGFzS0pEIDthbGtqIA==/"
</blockquote>

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

The IETF moq workgroup
