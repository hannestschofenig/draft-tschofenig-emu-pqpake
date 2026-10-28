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
  fullname: Hannes Tschofenig
  organization: University of the Bundeswehr Munich
  abbrev: UniBw M.
  city: Neubiberg
  country: Germany
  code: 85577
  email: hannes.tschofenig@gmx.net
-
  fullname: Tirumaleswar Reddy
  organization: Nokia
  email: kondtir@gmail.com
-
  fullname: Dan Harkins
  organization: Hewlett-Packard Enterprise
  email: daniel.harkins@hpe.com

normative:
  RFC2119:
  RFC8174:
  RFC3748:
  RFC5247:
  RFC5869:
  RFC8126:
  RFC7914:
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
CPaceOQUAKE+ from Section 8.3 of {{I-D.vos-cfrg-pqpake}}, revision
`draft-vos-cfrg-pqpake-03`.  CPaceOQUAKE+ is a hybrid
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
this document.  The registered verifier consists of `(v, pk, kem_blind)`;
the server also stores the salt, configuration, and registration
identities.  Registration MUST authenticate the server and protect the
confidentiality and integrity of the verifier material.

Either endpoint may derive the verifier: a server configured with the
password derives it and erases the password, or the peer derives it and
uploads it using the protected registration procedure described above.
The peer stores no persistent verifier state.

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
: The peer identity used when registering the CPaceOQUAKE+ verifier.

S:
: The server identity used when registering the CPaceOQUAKE+ verifier.

kem_blind:
: A secret, random 32-octet part of the registered verifier.  It blinds
  password-derived KEM key generation and is sent to the peer only in
  encrypted form during password confirmation.

public_context:
: Public session information encoded with `EncodePublicContext` as
  specified in Appendix C.1 of {{I-D.vos-cfrg-pqpake}}.

secret_context:
: Confidential context shared by the PAKE endpoints.  EAP-PQPAKE passes
  the empty octet string as this input to CPaceOQUAKE+.

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
CPaceOQUAKE+ is initiated by the client.  EAP-PQPAKE therefore uses a
server request carrying the salt and a fresh nonce to solicit the first
CPaceOQUAKE+ message from the peer after identity and configuration
negotiation have completed.

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
 EAP-PQPAKE-PAKE1/Response  (msg1: Ya)
 --------------------------------->

                          EAP-PQPAKE-PAKE2/Request
                          (msg2: Yb, s, T, rho)
                  <---------------------------------
 EAP-PQPAKE-PAKE2/Response  (msg3: ct, h)
 --------------------------------->

                          EAP-PQPAKE-Confirm/Request
                          (msg4: enc_c, client_confirm)
                  <---------------------------------
 EAP-PQPAKE-Confirm/Response  (msg5: server_confirm)
 --------------------------------->

                          EAP-Success
                  <---------------------------------
~~~

The Start exchange negotiates one CPaceOQUAKE+ configuration and
identifies the peer and server.  After receiving the peer identity, the
server locates the stored `(v, pk, kem_blind)` and salt for that peer
and selected configuration.  The PAKE1 and PAKE2 exchanges carry the
first three CPaceOQUAKE+ messages.  The Confirm exchange carries the
password confirmation challenge `msg4` and response `msg5`.  The field
`rho` in the diagram denotes the public matrix seed of ML-BUA-sKEM.

On success, both endpoints derive the EAP MSK and EMSK from the
CPaceOQUAKE+ method key.  On failure, either endpoint can send an
EAP-PQPAKE-Failure message.  The server terminates failed exchanges with
EAP-Failure.

# CPaceOQUAKE+ Use
{: #cpaceoquake-use}

EAP-PQPAKE uses the six CPaceOQUAKE+ functions in Section 8.3 of
{{I-D.vos-cfrg-pqpake}}.  The peer calls `Init`, `InitiatorContinue`,
and `InitiatorFinish`; the server calls `Respond`, `ResponderContinue`,
and `ResponderFinish`.

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
PRS, salt, U, and S using `GenVerifierMaterial`.  At registration,
`GenVerifier` creates `(v, pk, kem_blind)`: it samples a fresh
`kem_blind` and derives the KEM key pair from
`DeriveKEMSeed(seed, kem_blind)`.  The server MUST store `v`, `pk`,
`kem_blind`, the 32-octet salt, the configuration, and the registration
identities.  It MUST NOT store `seed` or the corresponding KEM secret
key.  The server MUST NOT disclose `kem_blind` in the Start or PAKE1
exchange; CPaceOQUAKE+ carries it only inside the encrypted password
confirmation challenge.

The EAP-PQPAKE session identifier `sid` and CPaceOQUAKE+ contexts are:

~~~ pseudocode
sid = H("EAP-PQPAKE sid" ||
        lv_encode(start_request_payload) ||
        lv_encode(start_response_payload) ||
        lv_encode(pake1_request_payload))
public_context = EncodePublicContext(sid, U, S)
secret_context = b""
~~~

Here `H` is the EAP hash selected by the configuration and `lv_encode`
is the two-octet length-prefix encoding defined in
{{I-D.vos-cfrg-pqpake}}.  Each `_payload` value is the complete
exchange-specific payload after reassembly, including its field length
prefixes.  It excludes the EAP header, Flags, PQPAKE-Exch, Total-Length,
and fragmentation acknowledgements.  Quoted labels are ASCII octets
without a terminating zero.

U is the Peer-ID value from Start/Response, excluding its type and
length; S is the Server-ID value from Start/Request, excluding its type
and length.  These MUST be the exact octet strings used for verifier
registration.  A peer MUST check S against its configured registration
before starting the PAKE.  Implementations MUST NOT normalize these
octet strings during the exchange.  Changing either registration
identity requires reprovisioning.  The identity types and their wire
encodings are also bound through `sid`.

`EncodePublicContext` MUST use the four-octet, big-endian length
prefixes from Appendix C.1 of {{I-D.vos-cfrg-pqpake}}.  Both endpoints
pass the resulting `public_context` and the empty `secret_context` to
their initial CPaceOQUAKE+ call.  CPaceOQUAKE internally supplies the
first stage's key and transcript hash as the contexts for OQUAKE.

The CPaceOQUAKE+ functions compute their own transcript hash `th`.
Password confirmation uses the `SK` and `th` output by the symmetric
CPaceOQUAKE stage: `PC.Challenge(SK, th, pk, kem_blind)` and
`PC.Respond(SK, th, seed, challenge)`.  Implementations MUST preserve
these inputs exactly; they MUST NOT substitute an EAP-specific
transcript hash for `th`.  The initial public context binds the EAP
configuration negotiation, identities, salt, and server nonce, while
the internal transcript hashes bind the subsequent PAKE messages.
The final `th` is returned with `client_key` or `server_key` after
successful password confirmation.

The intermediate `SK` MUST NOT be used for EAP key export.  The server
MUST NOT release or use the candidate key held in the state returned
by `ResponderContinue` until `ResponderFinish` succeeds.  The peer
obtains its method key only after `InitiatorFinish` verifies the
server's password confirmation value.

An implementation MUST abort the exchange if a CPaceOQUAKE+ operation
raises `AuthenticationError`, `CPaceError`, or another surfaced
cryptographic failure, if a received message has incorrect length or
framing for the selected configuration, or if the selected configuration
does not match one offered by the server.

OQUAKE's implicit rejection and random-key behavior MUST be preserved.
An implementation MUST NOT expose a distinguishable error for an
OQUAKE confirmation mismatch that the underlying protocol handles by
returning an unrelated key.  Such a mismatch is detected during the
subsequent password confirmation exchange.

# Configurations

An EAP-PQPAKE configuration identifies all cryptographic parameters
needed by CPaceOQUAKE+ and the EAP exporter.  This document defines one
mandatory-to-implement configuration:

`cpaceoquakeplus-x25519-mlbuaskem1024-xwing`:
: The preferred configuration in Section 9.5.1 of
  {{I-D.vos-cfrg-pqpake}}, with the common parameters from Section 9.1.

| Component | Algorithm or parameter |
|---|---|
| CPace group | CPACE-X25519-SHA512 |
| CPace hash | SHA-512 |
| OQUAKE BUA-sKEM | ML-BUA-sKEM-1024 |
| PAKE-KDF | HKDF-SHA-256 |
| Password-confirmation KEM | X-Wing |
| PC-KDF | HKDF-SHA-256 |
| PC-KSF | scrypt(N = 32768, r = 8, p = 1) |
| Common parameters | Nv = 32, Nkc = 32, Nr = 64, Nkey = 32 |
| EAP hash H and HKDF hash | SHA-256 |

The scrypt function is specified in {{RFC7914}}.  For ML-BUA-sKEM-1024,
Kemeleon.t is 132, Npk is 1596, Nt is 1564, the public matrix seed is
32 octets, and Nct is 1568.  For the X-Wing password-confirmation KEM,
Nseed is 32, Npk is 1216, and Nct is 1120.  This configuration requires
both ML-KEM-1024 for OQUAKE and ML-KEM-768 as used by X-Wing.

The domain separation tag is the following 32-octet value, shown in
hexadecimal:

~~~
7de162c387ba1bad9c790e7e56bd245d6753d045185cbc3972eba34ddac1a9db
~~~

This CPaceOQUAKE+ DST MUST be used throughout CPace, OQUAKE, transcript
hashing, verifier generation, and password confirmation.  It MUST NOT
be combined with the DST values of standalone CPaceOQUAKE or OQUAKE
configurations.

Implementations conforming to this document MUST implement this
configuration.  Other configurations MAY be registered as described in
{{iana}}.  New configurations MUST specify the complete CPaceOQUAKE+
parameter set, satisfy Section 9.6 of {{I-D.vos-cfrg-pqpake}}, and use
a hybrid KEM for password confirmation.  They MUST also specify the EAP
hash and the HKDF hash used by {{key-derivation}}.

This configuration and the `-03` message formats are incompatible with
the Ristretto255/Argon2id configuration and encodings in earlier
versions of this EAP draft.  Deployments using those versions MUST
reprovision verifiers with `GenVerifier` from the referenced revision,
including a fresh `kem_blind`.  The old verifier material MUST NOT be
reinterpreted as a registration for this configuration.

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
salt and verifier `(v, pk, kem_blind)` for its Server-ID.

## CPaceOQUAKE+ Message Mapping

EAP-PQPAKE carries the five CPaceOQUAKE+ protocol messages as EAP
payload fields.  Each field contains the complete serialized message
specified in Appendix C.5 of {{I-D.vos-cfrg-pqpake}}, inside an EAP
`opaque16` field.  This outer length prefix is additional to any length
prefix inside the serialized message.  The EAP exchanges map to the
CPaceOQUAKE+ functions as follows:

| EAP-PQPAKE field | Sender | Source value |
|---|---|---|
| CPaceOQUAKE-msg1 | peer | `msg1` from `CPaceOQUAKE+.Init` |
| CPaceOQUAKE-msg2 | server | `msg2` from `CPaceOQUAKE+.Respond` |
| CPaceOQUAKE-msg3 | peer | `msg3` from `CPaceOQUAKE+.InitiatorContinue` |
| PC-Challenge | server | `msg4` from `CPaceOQUAKE+.ResponderContinue` |
| PC-Response | peer | `msg5` from `CPaceOQUAKE+.InitiatorFinish` |

The following sequence uses the contexts and registration values from
{{cpaceoquake-use}}.  Each returned state replaces the endpoint's
previous state.  Serialization and parsing use Appendix C.5 of
{{I-D.vos-cfrg-pqpake}} at the EAP payload boundary.

~~~ pseudocode
peer_state, msg1 = CPaceOQUAKE+.Init(
    PRS, salt, U, S, public_context, secret_context)
server_state, msg2 = CPaceOQUAKE+.Respond(
    v, public_context, secret_context, msg1)
peer_state, msg3 = CPaceOQUAKE+.InitiatorContinue(
    peer_state, msg2)
server_state, msg4 = CPaceOQUAKE+.ResponderContinue(
    server_state, msg3, pk, kem_blind)
client_key, msg5, th = CPaceOQUAKE+.InitiatorFinish(
    peer_state, msg4)
server_key, th = CPaceOQUAKE+.ResponderFinish(server_state, msg5)
~~~

Before invoking a function that consumes a message, the receiver MUST
have reassembled the complete EAP payload and checked all field lengths
and framing.  Missing, duplicated, or additional fields MUST cause the
exchange to abort.  EAP fragmentation MUST NOT change field values,
message direction, or processing order, as required by Appendix D of
{{I-D.vos-cfrg-pqpake}}.  Retransmissions MUST reuse the original
encoded messages and cached EAP Responses; they MUST NOT cause a
CPaceOQUAKE+ operation to run again on the same state.

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
fresh random data generated by the server.  Salt is an opaque16 field
containing the 32-octet salt stored for the selected registration.

The PAKE1/Response payload is sent by the peer:

~~~ ascii-art
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       CPaceOQUAKE-msg1                       ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

CPaceOQUAKE-msg1 is an opaque16 field containing the first
CPaceOQUAKE+ message `msg1`, returned by `CPaceOQUAKE+.Init`.
Its contents are `lv_encode(Ya)`, where `Ya` is the CPace initiator
message.  The internal `lv_encode` prefix MUST be retained inside the
EAP opaque16 field.

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

CPaceOQUAKE-msg2 contains `msg2` from `CPaceOQUAKE+.Respond`, encoded
as `lv_encode(Yb) || s || T || rho`.  `Yb` is the CPace responder
message; `(s, T, rho)` is the OQUAKE initiator message.  CPaceOQUAKE-msg3
contains `msg3` from `CPaceOQUAKE+.InitiatorContinue`, encoded as
`ct || h`, the OQUAKE responder message.  The lengths of `s`, `T`,
`rho`, `ct`, and `h` are determined by the selected configuration.

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

Both fields are encoded as opaque16 values.  PC-Challenge contains
`msg4` from `CPaceOQUAKE+.ResponderContinue`, encoded as
`enc_c || client_confirm`.  `enc_c` encrypts the password-confirmation
KEM ciphertext together with `kem_blind`; its length is the
password-confirmation KEM's Nct plus 32 octets.  `client_confirm` has
Nkc octets.  PC-Response contains `msg5` from
`CPaceOQUAKE+.InitiatorFinish`, which is the Nkc-octet
`server_confirm`.  The confirmation names follow the underlying PAKE:
the server sends `client_confirm` and the peer sends `server_confirm`.

For the mandatory configuration, the serialized messages have the
following lengths, excluding the EAP opaque16 prefix and packet header:

| Message | Length in octets |
|---|---:|
| CPaceOQUAKE-msg1 | 34 |
| CPaceOQUAKE-msg2 | 1694 |
| CPaceOQUAKE-msg3 | 1600 |
| PC-Challenge | 1184 |
| PC-Response | 32 |

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

The peer obtains `client_key` and the final transcript hash `th` from
`CPaceOQUAKE+.InitiatorFinish` after verifying Confirm/Request.  The
server obtains the matching `server_key` and `th` from
`CPaceOQUAKE+.ResponderFinish` after verifying Confirm/Response.  These
keys are the method key used below.  No intermediate PAKE key or
unverified candidate key may be exported.  EAP-PQPAKE exports MSK and
EMSK as required by {{RFC3748}} and {{RFC5247}}.

The Method-ID is:

~~~ pseudocode
Method-ID = H("EAP-PQPAKE Method-ID" ||
              uint16(Selected-Config) || lv_encode(th))
~~~

`th` is the final CPaceOQUAKE+ transcript hash, which binds the
initial EAP public context and all five protocol messages, including
both password confirmation values.  It MUST NOT be replaced by the
transcript hash of an intermediate PAKE stage.  `Selected-Config` is
encoded as the two-octet value selected in Start/Response.

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

The MSK and EMSK are derived using HKDF {{RFC5869}} with the EAP HKDF
hash selected by the configuration (SHA-256 for the mandatory
configuration).  `Hash.length` is that hash's output length in octets:

~~~ pseudocode
exporter_secret = HKDF-Extract(zeros(Hash.length), method_key)
key_material = HKDF-Expand(exporter_secret,
                           "EAP-PQPAKE Exported Keys" ||
                           Session-ID ||
                           Method-ID,
                           128)
MSK  = key_material[0:64]
EMSK = key_material[64:128]
~~~

The slice end indexes are exclusive; MSK and EMSK are each 64 octets.

After deriving the MSK and EMSK, both endpoints MUST erase the
CPaceOQUAKE+ method key, intermediate PAKE secrets, KEM decapsulation
secrets, password-confirmation state, and ephemeral copies of the
password-derived seed and `kem_blind`.  This does not erase the server's
stored registration.  Ephemeral secrets and any unexported candidate
keys MUST also be erased when the method fails or times out.

# Security Considerations

EAP-PQPAKE inherits the cryptographic security properties and limitations
of CPaceOQUAKE+ {{I-D.vos-cfrg-pqpake}}.  The EAP encapsulation defined
here binds EAP identities, configuration negotiation, salt, server nonce,
and all carried CPaceOQUAKE+ messages through its public context and
internal transcript hashes.  The following claims are stated in the
style expected for EAP methods used in wireless LANs {{RFC4017}}.

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

Section 11.2 of {{I-D.vos-cfrg-pqpake}} qualifies the post-quantum
security analysis: the UC proofs use the classical random oracle model.
OQUAKE-style protocols do not realize the standard UC PAKE
functionality with quantum random-oracle access under the cited
analysis, although this does not yield an attack recovering passwords
or keys and a closely related protocol has a stand-alone game-based
proof in that model.  EAP-PQPAKE does not claim a stronger quantum UC
result.  Its security against quantum attackers relies on OQUAKE;
CPace alone does not provide post-quantum PAKE security.

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
  malformed encodings, and configuration mismatches, while preserving
  OQUAKE's implicit rejection behavior described in {{cpaceoquake-use}}.

Dictionary attacks:
: Password authentication methods cannot prevent online guessing.  EAP
  servers MUST implement rate limits or equivalent controls for failed
  authentications.  Implementations SHOULD avoid distinguishable error
  behavior between unknown identities and incorrect passwords.

Forward secrecy:
: Compromise of the password after an exchange completes does not reveal
  earlier MSK or EMSK values unless the attacker also breaks the
  underlying assumptions for the recorded exchange.  Compromise of the
  full registered verifier `(v, pk, kem_blind)` permits offline password
  guessing and server impersonation to the peer.  It does not by itself
  permit client impersonation without recovering the password, the
  password-derived seed, or the corresponding decapsulation key, as
  discussed in Section 11.5 of {{I-D.vos-cfrg-pqpake}}.  Affected
  credentials require reprovisioning.

Downgrade protection:
: The selected configuration and the server's offered configuration list
  are included in the EAP-PQPAKE transcript.  A downgrade attempt that
  changes negotiation messages is detected during password confirmation.
  Implementations MUST NOT offer configurations that are prohibited by
  local policy.

Identity protection:
: EAP-PQPAKE does not encrypt the Start exchange.  A passive observer can
  see the server and peer identities unless this method is used inside a
  protected tunnel.  An anonymous outer EAP identity alone does not hide
  the identities in the Start exchange.  This mapping uses the same
  identities for registration and the session public context; the
  underlying PAKE permits these to differ, but this document does not
  define such an identity mapping.  Changing a registration identity
  requires recomputing and reprovisioning the verifier.

Verifier handling:
: The server stores the salt and the CPaceOQUAKE+ verifier
  `(v, pk, kem_blind)`.  The password-derived seed used to derive the
  password-confirmation KEM key pair MUST NOT be stored by the server.
  Registration and reprovisioning are outside this method
  and need channel authentication, confidentiality, and integrity.  A
  fresh secret `kem_blind` is required for each registration and MUST
  be protected with the rest of the verifier.  The memory-hard password
  stretching used by CPaceOQUAKE+ hardens
  password-derived verifier material against offline guessing after
  verifier compromise; it does not replace online rate limiting for EAP
  authentication attempts.

Side channels:
: CPaceOQUAKE+ discusses timing considerations for OQUAKE and ML-KEM
  processing.  In addition to the Tempo construction used by OQUAKE,
  the augmented transformation blinds KEM key generation with
  `kem_blind`.  Removing this blinding or disclosing `kem_blind` before
  password confirmation invalidates that protection.  Implementations
  MUST preserve the blinding and its encryption, use constant-time
  comparison for confirmation values, and keep Kemeleon.EncodeEk
  randomness secret.  They SHOULD use implementations of the selected
  primitives that address known timing attacks.

Denial of service:
: The recommended configuration uses scrypt password stretching on the
  peer and post-quantum cryptography on both endpoints.  Servers SHOULD
  apply admission control before expensive operations, for example by
  limiting concurrent sessions per peer, realm, or lower-layer attachment.

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
| 0x0001 | cpaceoquakeplus-x25519-mlbuaskem1024-xwing | RFC-TBD |

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
