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
  CAT:
    title: "CTA 5007-A Common Access Token"
    date: December 2024
    target: [https://www.w3.org/TR/webcodecs-codec-registry/](https://shop.cta.tech/products/cta-5007)

informative:


--- abstract

A token-based authentication scheme for use with Media Over QUIC Transport.


--- middle

# Introduction

This draft introduces a token-based authentication scheme for use with MOQT {{MoQTransport}}.
The scheme protects access to the server during session establishment and also contrains the
actions which the client may take once connected.


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
