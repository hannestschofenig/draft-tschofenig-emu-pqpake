---
title: "The EAP-PQPAKE Authentication Method"
abbrev: "EAP-PQPAKE"
category: exp

docname: draft-tschofenig-emu-pqpake-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "EAP Method Update"
keyword:
 - EAP
 - PAKE
 - post-quantum
 - password authentication
venue:
  group: "EAP Method Update"
  type: "Working Group"
  mail: "emu@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/emu"
  github: "hannestschofenig/draft-tschofenig-emu-pqpake"
  latest: "https://hannestschofenig.github.io/draft-tschofenig-emu-pqpake/draft-tschofenig-emu-pqpake.html"

author:
-
  ins: H. Tschofenig
  name: Hannes Tschofenig
  organization: University of the Bundeswehr Munich
  abbrev: UniBw M.
  city: Neubiberg
  region: Bavaria
  country: Germany
  code: 85577
  email: hannes.tschofenig@gmx.net

normative:
  RFC2119:
  RFC8174:
  RFC3748:
  RFC5247:
  RFC5869:
  RFC8126:
  RFC9106:
  I-D.vos-cfrg-pqpake:

informative:
  RFC4017:
  RFC4962:
  RFC5931:
  RFC6124:

...

--- abstract

This document defines EAP-PQPAKE, an Extensible Authentication Protocol
(EAP) method that uses a hybrid post-quantum password-authenticated key
exchange.  The method is based on the CPaceOQUAKE+ asymmetric PAKE and
provides mutual authentication and EAP key export from a password without
requiring public-key certificates during normal authentication.

--- middle

# Introduction

The Extensible Authentication Protocol (EAP) {{RFC3748}} supports
multiple authentication methods between an EAP peer and an EAP server.
Several EAP methods use passwords directly, including EAP-pwd {{RFC5931}}
and EAP-EKE {{RFC6124}}.  These methods address the operational need for
authentication from a human-memorable password while limiting the value
of passive observation and offline dictionary attacks.

Existing password-based EAP methods are built from classical public-key
assumptions.  A quantum-capable attacker that records authentication
exchanges today may later be able to recover keying material when those
classical assumptions fail.  This is undesirable for deployments where
the confidentiality or integrity of subsequent traffic remains valuable
after the authentication exchange has completed.

This document specifies EAP-PQPAKE, an EAP method that encapsulates
CPaceOQUAKE+ {{I-D.vos-cfrg-pqpake}}.  CPaceOQUAKE+ is a hybrid
asymmetric PAKE that combines a classical PAKE component with a
post-quantum KEM-based component.  EAP-PQPAKE defines the EAP message
flow, packet formats, method transcript, exported MSK and EMSK
derivation, fragmentation behavior, and IANA registries needed to use
CPaceOQUAKE+ as an EAP method.

CPaceOQUAKE+ is derived from CPaceOQUAKE, a hybrid symmetric PAKE that
sequentially composes CPace and OQUAKE.  CPace contributes a classical
PAKE whose transcript is unconditionally password hiding, while OQUAKE
contributes a post-quantum KEM-based PAKE component.

EAP-PQPAKE assumes that the server has been provisioned with the
CPaceOQUAKE+ password verifiers for the peer.  The registration
procedure used to create those verifiers is specified by
CPaceOQUAKE+ and is outside the EAP authentication exchange defined in
this document.  Registration must be performed over a channel that
authenticates the server and protects the verifier material from
modification.

Note: This document version is experimental and relies on a
work in progress PAKE algorithm.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following terms are used throughout this document:

EAP peer:
: The endpoint being authenticated by the EAP server.

EAP server:
: The endpoint that selects the EAP method, stores the peer's
  CPaceOQUAKE+ verifier state, and authenticates the peer.

PRS:
: The password-related string input to CPaceOQUAKE+.  This is commonly
  the password after local password processing.

U:
: The peer identity input to CPaceOQUAKE+.

S:
: The server identity input to CPaceOQUAKE+.

Config:
: A registered set of CPaceOQUAKE+ algorithms and parameters.

Method key:
: The 32-octet `client_key` or `server_key` output by the CPaceOQUAKE+
  password confirmation stage.  EAP-PQPAKE expands this value to the
  MSK and EMSK.

uint8, uint16, uint32:
: Unsigned integers encoded in network byte order.

opaque16:
: A variable-length octet string encoded as a uint16 length followed by
  that many octets.

# Protocol Overview

EAP is a lock-step request/response protocol in which the EAP server
sends each Request and the EAP peer sends the corresponding Response.
CPaceOQUAKE+ is initiated by the client.  EAP-PQPAKE therefore uses an
empty server request to solicit the first CPaceOQUAKE+ flight from the
peer after identity and configuration negotiation have completed.

A successful EAP-PQPAKE exchange has the following shape:

~~~ ascii-art
 EAP peer                                      EAP server
 --------                                      ----------

                          EAP-PQPAKE-Start/Request
                  <---------------------------------
 EAP-PQPAKE-Start/Response
 --------------------------------->

                          EAP-PQPAKE-PAKE1/Request
                  <---------------------------------
 EAP-PQPAKE-PAKE1/Response  (s1, msg1)
 --------------------------------->

                          EAP-PQPAKE-PAKE2/Request
                          (s2, msg2, msg3)
                  <---------------------------------
 EAP-PQPAKE-PAKE2/Response  (msg4)
 --------------------------------->

                          EAP-PQPAKE-Confirm/Request (chal)
                  <---------------------------------
 EAP-PQPAKE-Confirm/Response  (resp)
 --------------------------------->

                          EAP-Success
                  <---------------------------------
~~~

The Start exchange negotiates one CPaceOQUAKE+ configuration and
identifies the peer and server.  After receiving the peer identity, the
server locates the stored verifier, password-confirmation KEM public
key, and salt for that peer and selected configuration.  The PAKE1 and
PAKE2 exchanges carry the three CPaceOQUAKE flights: `(s1, msg1)`,
`(s2, msg2, msg3)`, and `msg4`.  The Confirm exchange carries the
CPaceOQUAKE+ password confirmation messages `chal` and `resp`.

On success, both endpoints derive the EAP MSK and EMSK from the
CPaceOQUAKE+ method key.  On failure, either endpoint can send an
EAP-PQPAKE-Failure message.  The server terminates failed exchanges with
EAP-Failure.

# CPaceOQUAKE+ Use
{: #cpaceoquake-use}

EAP-PQPAKE uses CPaceOQUAKE+ as specified in {{I-D.vos-cfrg-pqpake}},
with the following mapping.

The CPaceOQUAKE+ messages carried by this method are opaque to
EAP-PQPAKE.  Implementations that implement CPaceOQUAKE+ directly, rather
than using a CPaceOQUAKE+ library interface, MUST follow the selected
CPaceOQUAKE+ configuration exactly.  In particular, OQUAKE ciphertext
lengths and parsing are determined by the BUA-sKEM parameters of that
configuration and MUST NOT be inferred from public-key lengths.  Kemeleon
uniform encoding is applied only where CPaceOQUAKE+ specifies it, such as
to the ML-BUA-sKEM public-key material, and MUST NOT be applied to
ciphertexts unless the selected CPaceOQUAKE+ configuration explicitly
requires it.

The peer computes the CPaceOQUAKE+ verifier material `(v, seed)` from
PRS, salt, U, and S using `GenVerifierMaterial`.  The server stores the
salt, verifier `v`, and password-confirmation KEM public key `pk` for
the peer.  The server MUST NOT store the seed that CPaceOQUAKE+ derives
from the password.

The CPaceOQUAKE+ session identifier `sid` is:

~~~ pseudocode
sid = H("EAP-PQPAKE sid" ||
        start_request_payload ||
        start_response_payload ||
        pake1_request_payload)
~~~

where `H` is the hash function selected by the EAP-PQPAKE configuration.
The peer identity U is the Identity field from the Start/Response.  The
server identity S is the Identity field from the Start/Request.  Both
identities are the exact encoded octet strings sent on the wire, not a
display-normalized form.

The CPaceOQUAKE+ password-confirmation transcript `tx` is:

~~~ pseudocode
tx = H("EAP-PQPAKE transcript" ||
       start_request_payload ||
       start_response_payload ||
       pake1_request_payload ||
       pake1_response_payload ||
       pake2_request_payload ||
       pake2_response_payload)
~~~

The hash input includes the complete EAP-PQPAKE payloads, excluding EAP
Code, Identifier, Length, Type, fragmentation flags, Total-Length, and
fragmentation acknowledgements.  This binds the selected configuration,
identities, salt, server nonce, and all CPaceOQUAKE messages into the
password confirmation stage.  This value is used as the `transcript`
input to the CPaceOQUAKE+ `PC-Challenge` and `PC-Response` functions.

An implementation MUST abort the exchange if any CPaceOQUAKE+ operation
raises an authentication error, if a received cryptographic message is
not valid for the selected configuration, or if the selected
configuration does not match one offered by the server.

# Configurations

An EAP-PQPAKE configuration identifies all cryptographic parameters
needed by CPaceOQUAKE+ and the EAP exporter.  This document defines one
mandatory-to-implement configuration:

PQPAKE-XWING-MLBUASKEM1024-CPACE-RISTRETTO255-HKDFSHA256-ARGON2ID:
: This configuration uses the recommended CPaceOQUAKE+ configuration
  from {{I-D.vos-cfrg-pqpake}}: CPACE-RISTR255-SHA512, X-Wing for the
  password-confirmation KEM, HKDF-SHA-256 for the
  password-confirmation KDF, Argon2id {{RFC9106}} with the parameters
  specified by CPaceOQUAKE+, ML-BUA-sKEM1024 for OQUAKE, HKDF-SHA-256
  for the PAKE KDF, SHA-256 as the general-purpose hash, and the
  CPaceOQUAKE+ recommended domain separation tag.  The ML-BUA-sKEM1024
  OQUAKE parameters, including Kemeleon security parameter and public-key
  and ciphertext lengths, are those specified by CPaceOQUAKE+ for this
  configuration.

Implementations conforming to this document MUST implement this
configuration.  Other configurations MAY be registered as described in
{{iana}}.  New configurations MUST specify the complete CPaceOQUAKE+
parameter set and MUST use a hybrid KEM for the password-confirmation
KEM, as required by CPaceOQUAKE+.

# Packet Format

EAP-PQPAKE packets consist of the standard EAP header followed by an
EAP-PQPAKE method header and the exchange-specific payload.  Multi-octet
integers are encoded in network byte order.

~~~ ascii-art
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Code      |  Identifier   |             Length            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type      |     Flags     |  PQPAKE-Exch  |   Data ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

Code, Identifier, Length, and Type are the EAP fields defined in
{{RFC3748}}.  The Type value for EAP-PQPAKE is TBD1.

The Flags field is defined as follows:

~~~ ascii-art
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
|L|M| Reserved  |
+-+-+-+-+-+-+-+-+
~~~

L:
: Length included.  When set, a uint16 Total-Length field appears at the
  start of Data.

M:
: More fragments.  When set, more fragments of the same EAP-PQPAKE
  message follow.

Reserved:
: MUST be sent as zero and MUST be ignored by the receiver.

PQPAKE-Exch identifies the exchange payload:

| Value | Exchange |
|---:|---|
| 0x00 | Reserved |
| 0x01 | Start |
| 0x02 | PAKE1 |
| 0x03 | PAKE2 |
| 0x04 | Confirm |
| 0x05 | Failure |
| 0x06-0x7f | Unassigned |
| 0x80-0xff | Private Use |

If the L bit is set, Data begins with:

~~~ ascii-art
 0                   1
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Total-Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

Total-Length is the length in octets of the unfragmented EAP-PQPAKE
message, including the EAP header, EAP-PQPAKE method header, and
payload.

## Identity Types

Identity Type is a one-octet field.  This document defines the following
values:

| Value | Name | Meaning |
|---:|---|---|
| 0x00 | Reserved | Reserved |
| 0x01 | Opaque | Opaque octet string |
| 0x02 | NAI | Network Access Identifier |
| 0x03 | FQDN | Fully Qualified Domain Name |
| 0x04-0x7f | Unassigned | Available for assignment |
| 0x80-0xff | Private Use | Private Use |

Implementations MUST support receiving Opaque and NAI identities.
Implementations that use FQDN identities MUST use the A-label form for
internationalized domain names.

## Start Payloads

The Start/Request payload is sent by the server:

~~~ ascii-art
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Config-Count  |   Reserved    |        Config-ID[0]          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Config-ID[1] ...                                      ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Server-IDType |              Server-ID                       ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

Config-Count is the number of 16-bit Config-ID values offered by the
server.  It MUST be greater than zero.  Config-ID values are listed in
preference order.  Reserved MUST be sent as zero and ignored by the
receiver.  Server-ID is an opaque16 field preceded by Server-IDType.

The Start/Response payload is sent by the peer:

~~~ ascii-art
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Selected-Config        | Peer-IDType   | Peer-ID      ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

Selected-Config MUST be one of the Config-ID values in the
Start/Request.  Peer-ID is an opaque16 field preceded by Peer-IDType.
The server uses Selected-Config and Peer-ID to locate the peer's stored
salt, verifier, and password-confirmation KEM public key.

## CPaceOQUAKE+ Message Mapping

EAP-PQPAKE carries the five CPaceOQUAKE+ protocol messages as EAP
payload fields.  The first three fields carry the encoded messages of
the CPaceOQUAKE subprotocol inside CPaceOQUAKE+.  The EAP exchanges map
to the CPaceOQUAKE+ functions in {{I-D.vos-cfrg-pqpake}} as follows:

| EAP-PQPAKE field | Sender | Source value |
|---|---|---|
| CPaceOQUAKE-msg1 | peer | `(s1, msg1)` from `CPaceOQUAKE.Init` |
| CPaceOQUAKE-msg2 | server | `(s2, msg2, msg3)` from `CPaceOQUAKE.Respond` |
| CPaceOQUAKE-msg3 | peer | `msg4` from `CPaceOQUAKE.InitiatorFinish` |
| PC-Challenge | server | `challenge` from `PC-Challenge` |
| PC-Response | peer | `response` from `PC-Response` |

Here, `v` is the CPaceOQUAKE+ verifier for the selected peer and
configuration.  The peer computes `v` and `seed` from PRS, salt, U, and
S using `GenVerifierMaterial`; the server uses the stored `v` and
password-confirmation KEM public key `pk` for the peer.  `pk` is
generated during CPaceOQUAKE+ registration.  `SK` is the symmetric key
output by the CPaceOQUAKE phase before password confirmation, and `tx`
is the EAP-PQPAKE password-confirmation transcript defined in
{{cpaceoquake-use}}.

The tuple notation in this table follows the CPaceOQUAKE flow in
{{I-D.vos-cfrg-pqpake}}.  On the wire, CPaceOQUAKE-msg1 is the encoded
`init_msg` returned by `CPaceOQUAKE.Init`, consisting of `s1` followed
by `lv_encode(msg1)`.  CPaceOQUAKE-msg2 is the encoded `resp_msg`
returned by `CPaceOQUAKE.Respond`, consisting of `s2` followed by
`lv_encode(msg2)` and `lv_encode(msg3)`.  CPaceOQUAKE-msg3 is the
`msg4` returned by `CPaceOQUAKE.InitiatorFinish`.  `PC-Challenge` is
invoked with `SK`, `tx`, `pk`, `sid`, U, and S.  `PC-Response` is
invoked with `SK`, `tx`, `seed`, `challenge`, `sid`, U, and S.

## PAKE1 Payloads

The PAKE1/Request payload is sent by the server:

~~~ ascii-art
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Server-Nonce                         ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             Salt                              ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

Server-Nonce is an opaque16 field containing at least 32 octets of
fresh random data generated by the server.  Salt is the opaque16 salt
stored for the selected peer and configuration.

The PAKE1/Response payload is sent by the peer:

~~~ ascii-art
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       CPaceOQUAKE-msg1                       ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

CPaceOQUAKE-msg1 is an opaque16 field containing the first
CPaceOQUAKE+ message.  It is the encoded `init_msg` returned by
`CPaceOQUAKE.Init`, corresponding to `(s1, msg1)` in
{{I-D.vos-cfrg-pqpake}}.

## PAKE2 Payloads

The PAKE2/Request payload is sent by the server:

~~~ ascii-art
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       CPaceOQUAKE-msg2                       ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The PAKE2/Response payload is sent by the peer:

~~~ ascii-art
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       CPaceOQUAKE-msg3                       ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

Both fields are encoded as opaque16 values.

CPaceOQUAKE-msg2 is the encoded `resp_msg` returned by
`CPaceOQUAKE.Respond`, corresponding to `(s2, msg2, msg3)` in
{{I-D.vos-cfrg-pqpake}}.  CPaceOQUAKE-msg3 is the `msg4` returned by
`CPaceOQUAKE.InitiatorFinish`.

## Confirm Payloads

The Confirm/Request payload is sent by the server:

~~~ ascii-art
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         PC-Challenge                         ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The Confirm/Response payload is sent by the peer:

~~~ ascii-art
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          PC-Response                         ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

Both fields are encoded as opaque16 values.  PC-Challenge is the
challenge output by CPaceOQUAKE+ `PC-Challenge`.  PC-Response is the
response output by CPaceOQUAKE+ `PC-Response`.

## Failure Payload

The Failure payload is a uint32 Failure-Code:

| Value | Name | Meaning |
|---:|---|---|
| 0x00000000 | Reserved | Reserved |
| 0x00000001 | No Error | Failure acknowledgement |
| 0x00000002 | Protocol Error | Malformed or unexpected message |
| 0x00000003 | Identity Unknown | No verifier is available for the identity |
| 0x00000004 | Authentication Failure | Password confirmation or PAKE validation failed |
| 0x00000005 | Authorization Failure | Authentication succeeded but access is denied |
| 0x00000006 | No Config Chosen | No acceptable configuration was selected |
| 0x00000007-0xfeffffff | Unassigned | Available for assignment |
| 0xff000000-0xffffffff | Private Use | Private Use |

When the peer detects an error, it SHOULD send Failure/Response.  The
server MUST then send EAP-Failure.  When the server detects an error, it
SHOULD send Failure/Request.  The peer SHOULD answer with
Failure/Response containing No Error, and the server MUST then send
EAP-Failure.  Implementations MAY use Authentication Failure instead of
Identity Unknown to avoid disclosing whether a peer identity exists.

# Fragmentation

Post-quantum KEM public keys and ciphertexts can make EAP-PQPAKE
messages larger than a lower layer's EAP payload limit.  EAP-PQPAKE
therefore supports fragmentation using the L and M flags in the method
header.

Implementations MUST support receiving fragmented EAP-PQPAKE messages.
An implementation MUST establish a fragmentation threshold.  If the lower
layer MTU is known, the threshold SHOULD be derived from that MTU.  If no
lower-layer limit is known, the threshold MUST be 1020 octets.

The first fragment of a fragmented message MUST set L and M and MUST
include Total-Length.  Intermediate fragments MUST set M and MUST NOT set
L.  The final fragment MUST NOT set M and MUST NOT set L.  A
non-fragmented message MUST NOT set L or M.

When the peer receives an EAP-PQPAKE Request fragment with M set, it
MUST respond with an EAP-PQPAKE Response fragment acknowledgement.  The
acknowledgement uses the same PQPAKE-Exch value, has no payload, and has
L and M cleared.  When the server receives an EAP-PQPAKE Response
fragment with M set, it MUST respond with an EAP-PQPAKE Request fragment
acknowledgement with the same properties.  The next fragment is sent only
after the acknowledgement is received.

Receivers MUST reassemble all fragments before parsing the
exchange-specific payload.  A receiver MUST abort if Total-Length is
inconsistent with the received fragments, if fragments from different
exchanges are interleaved, or if the reassembled message exceeds a local
policy limit.

# Key Derivation

After successful processing of Confirm/Response, both endpoints have the
same CPaceOQUAKE+ method key.  EAP-PQPAKE exports MSK and EMSK as
required by {{RFC3748}} and {{RFC5247}}.

The Method-ID is:

~~~ pseudocode
Method-ID = H("EAP-PQPAKE Method-ID" ||
              selected_config ||
              start_request_payload ||
              start_response_payload ||
              pake1_request_payload ||
              pake1_response_payload ||
              pake2_request_payload ||
              pake2_response_payload ||
              confirm_request_payload ||
              confirm_response_payload)
~~~

The Session-ID is:

~~~ pseudocode
Session-ID = EAP-Type-Code || Method-ID
~~~

where EAP-Type-Code is the single-octet EAP method type assigned by
IANA for EAP-PQPAKE.

Following the key naming guidance in {{RFC4962}}, exported key names are:

~~~ pseudocode
MSK-name  = Session-ID || "MSK"
EMSK-name = Session-ID || "EMSK"
~~~

The MSK and EMSK are derived using HKDF {{RFC5869}} with the hash
function selected by the configuration:

~~~ pseudocode
exporter_secret = HKDF-Extract(zeros(Hash.length), method_key)
key_material = HKDF-Expand(exporter_secret,
                           "EAP-PQPAKE Exported Keys" ||
                           Session-ID ||
                           Method-ID,
                           128)
MSK  = key_material[0..63]
EMSK = key_material[64..127]
~~~

After deriving the MSK and EMSK, both endpoints MUST erase the
CPaceOQUAKE+ method key, intermediate PAKE secrets, KEM decapsulation
secrets, password-confirmation context, and any password-derived seed.
The same values MUST be erased when the method fails or times out.

# Security Considerations

EAP-PQPAKE inherits the cryptographic security properties and limitations
of CPaceOQUAKE+ {{I-D.vos-cfrg-pqpake}}.  The EAP encapsulation defined
here binds EAP identities, configuration negotiation, salt, server nonce,
and all carried CPaceOQUAKE+ messages into the CPaceOQUAKE+ session
identifier or password-confirmation transcript.  The following claims
are stated in the style expected for EAP methods used in wireless LANs
{{RFC4017}}.

The hybrid security of CPaceOQUAKE+ depends on the CPaceOQUAKE+
composition specified in {{I-D.vos-cfrg-pqpake}}.  The sequential
composition relies on CPace being unconditionally password hiding: the
CPace transcript does not reveal information about the password-related
string even if the classical Diffie-Hellman assumption later fails.
OQUAKE is not assumed to have that unconditional password-hiding
property; it provides the post-quantum KEM-based component of the
hybrid design.  EAP-PQPAKE therefore does not define a parallel
composition and does not permit substituting a different PAKE component
unless that component is specified by a registered CPaceOQUAKE+
configuration with an applicable security analysis.

Passive attacks:
: A passive attacker observing EAP-PQPAKE messages is not expected to
  learn the password, the CPaceOQUAKE+ method key, the MSK, or the EMSK.
  The method is intended to protect recorded exchanges against attackers
  with future quantum capabilities, subject to the hybrid security
  assumptions of CPaceOQUAKE+.

Active attacks:
: An active attacker can drop, delay, replay, or modify EAP messages.
  Such modification causes transcript or confirmation failure unless the
  attacker knows the password or breaks the underlying CPaceOQUAKE+
  assumptions.  Implementations MUST reject unexpected exchange ordering,
  malformed encodings, and configuration mismatches.

Dictionary attacks:
: Password authentication methods cannot prevent online guessing.  EAP
  servers MUST implement rate limits or equivalent controls for failed
  authentications.  Implementations SHOULD avoid distinguishable error
  behavior between unknown identities and incorrect passwords.

Forward secrecy:
: Compromise of the password after an exchange completes does not reveal
  earlier MSK or EMSK values unless the attacker also breaks the
  underlying assumptions for the recorded exchange.  Server compromise
  that reveals stored verifiers and password-confirmation KEM public
  keys can enable impersonation according to the CPaceOQUAKE+ security
  model and requires credential reprovisioning.

Downgrade protection:
: The selected configuration and the server's offered configuration list
  are included in the EAP-PQPAKE transcript.  A downgrade attempt that
  changes negotiation messages is detected during password confirmation.
  Implementations MUST NOT offer configurations that are prohibited by
  local policy.

Identity protection:
: EAP-PQPAKE does not encrypt the Start exchange.  A passive observer can
  see the server and peer identities unless this method is used inside a
  protected tunnel or the deployment uses privacy-preserving outer
  identities.  Since identities are inputs to CPaceOQUAKE+ verifier
  generation, deployments need a stable identity plan before
  provisioning verifiers.

Verifier handling:
: The server stores the CPaceOQUAKE+ verifier, salt, and
  password-confirmation KEM public key.  The password-derived seed used
  to derive the password-confirmation KEM key pair MUST NOT be stored by
  the server.  Registration and reprovisioning are outside this method
  and need channel authentication and integrity protection.  The
  memory-hard password stretching used by CPaceOQUAKE+ hardens
  password-derived verifier material against offline guessing after
  verifier compromise; it does not replace online rate limiting for EAP
  authentication attempts.

Side channels:
: CPaceOQUAKE+ discusses timing considerations for OQUAKE and ML-KEM
  processing.  EAP-PQPAKE implementations MUST use constant-time
  comparison for confirmation values and SHOULD use implementations of
  the selected primitives that address known timing attacks.

Denial of service:
: The recommended configuration uses memory-hard password stretching and
  post-quantum cryptography.  Servers SHOULD apply admission control
  before expensive operations, for example by limiting concurrent
  sessions per peer, realm, or lower-layer attachment.

Fragmentation:
: Fragmentation increases buffering requirements.  Implementations MUST
  enforce local maximum reassembled lengths and discard incomplete
  fragmented messages on timeout.

# IANA Considerations
{: #iana}

IANA is requested to allocate one EAP Method Type from the "Method Types"
registry:

| Value | Description | Reference |
|---:|---|---|
| TBD1 | EAP-PQPAKE | RFC-TBD |

IANA is requested to create the "EAP-PQPAKE Exchange Types" registry.
The registration policy is Specification Required {{RFC8126}} for values
0x06 through 0x7f.  Values 0x80 through 0xff are for Private Use.  The
initial values are listed in {{packet-format}}.

IANA is requested to create the "EAP-PQPAKE Configurations" registry.
The registry contains 16-bit values.  The registration policy is
Specification Required for values 0x0002 through 0x7fff.  Values 0x8000
through 0xffff are for Private Use.  The initial contents are:

| Value | Name | Reference |
|---:|---|---|
| 0x0000 | Reserved | RFC-TBD |
| 0x0001 | PQPAKE-XWING-MLBUASKEM1024-CPACE-RISTRETTO255-HKDFSHA256-ARGON2ID | RFC-TBD |

IANA is requested to create the "EAP-PQPAKE Identity Types" registry.
The registry contains 8-bit values.  The registration policy is
Specification Required for values 0x04 through 0x7f.  Values 0x80
through 0xff are for Private Use.  The initial values are listed in
{{identity-types}}.

IANA is requested to create the "EAP-PQPAKE Failure Codes" registry.  The
registry contains 32-bit values.  The registration policy is
Specification Required for values 0x00000007 through 0xfeffffff.  Values
0xff000000 through 0xffffffff are for Private Use.  The initial values
are listed in {{failure-payload}}.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge contributors and reviewers.
