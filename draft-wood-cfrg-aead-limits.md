---
title: Limits on AEAD Algorithms
abbrev: AEAD Limits
docname: draft-wood-cfrg-aead-limits-latest
date:
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -  ins: M. Thomson
    name: Martin Thomson
    org: Mozilla
    email: mt@lowentropy.net
 -  ins: C. A. Wood
    name: Christopher A. Wood
    org: Cloudflare
    email: caw@heapingbits.net

normative:
  GCM:
    title: "Recommendation for Block Cipher Modes of Operation: Galois/Counter Mode (GCM) and GMAC"
    date: 2007-11
    author:
      - ins: M. Dworkin
    seriesinfo:
      NIST: Special Publication 800-38D

informative:
  NonceDisrespecting:
    target: https://eprint.iacr.org/2016/475.pdf
    title: "Nonce-Disrespecting Adversaries -- Practical Forgery Attacks on GCM in TLS"
    author:
      - ins: H. Bock
      - ins: A. Zauner
      - ins: S. Devlin
      - ins: J. Somorovsky
      - ins: P. Jovanovic
    date: 2016-05-17
  Poly1305:
    title: "The Poly1305-AES message-authentication code"
    target: https://link.springer.com/content/pdf/10.1007/11502760_3.pdf
    author:
      - ins: D. J. Bernstein
    seriesinfo: "International Workshop on Fast Software Encryption, 2005"
  ChaCha20Poly1305Bounds:
    title: "A Security Analysis of the Composition of ChaCha20 and Poly1305"
    author:
      - ins: G. Procter
    date: 2014
    target: https://eprint.iacr.org/2014/613.pdf
  AEBounds:
    title: "Limits on Authenticated Encryption Use in TLS"
    author:
      - ins: A. Luykx
      - ins: K. Paterson
    date: 2016-03-08
    target: http://www.isg.rhul.ac.uk/~kp/TLS-AEbounds.pdf

--- abstract

An Authenticated Encryption with Associated Data (AEAD) algorithm provides
confidentiality and integrity.  Excessive use of the same key can give an
attacker advantages in breaking these properties.  This document provides simple
guidance for users of common AEAD functions about how to limit the use of keys
in order to bound the advantage given to an attacker.

--- middle

# Introduction

An Authenticated Encryption with Associated Data (AEAD) algorithm
provides confidentiality and integrity. {{!RFC5116}} specifies an AEAD
as a function with four inputs -- secret key, nonce, plaintext,
and optional associated data -- that produces ciphertext output and error code
indicating success or failure. The ciphertext is typically composed of the encrypted
plaintext bytes and an authentication tag.

The generic AEAD interface does not describe usage limits.  Each AEAD algorithm
does describe limits on its inputs, but these are formulated as strict
functional limits, such as the maximum length of inputs, which are determined by
the properties of the underlying AEAD composition.  Degradation of the security
of the AEAD as a single key is used multiple times is not given a thorough
treatment.

The number of times a single pair of key and nonce can be used might also be
relevant to security.  For some algorithms, such as AEAD_AES_128_GCM or
AEAD_AES_128_GCM, this limit is 1 and using the same pair of key and nonce has
serious consequences for both confidentiality and integrity; see
{{NonceDisrespecting}}.  Nonce-reuse resistant algorithms like
AEAD_AES_128_GCM_SIV can tolerate a limited amount of nonce reuse.

It is good practice to have limits on how many times the same key (or pair of
key and nonce) are used.  Setting a limit based on some measurable property of
the usage, such as number of protected messages or amount of data transferred,
ensures that it is easy to apply limits.  This might require the application of
simplifying assumptions.  For example, TLS 1.3 specifies limits on the number of
records that can be protected, using the simplifying assumption that records are
the same size; see Section 5.5 of {{?TLS=RFC8446}}.

Currently, AEAD limits and usage requirements are scattered among peer-reviewed
papers, standards documents, and other RFCs. Determining the correct limits for
a given setting is challenging as papers do not use consistent labels or
conventions, and rarely apply any simplifications that might aid in reaching a
simple limit.

The intent of this document is to collate all relevant information about the
proper usage and limits of AEAD algorithms in one place.  This may serve as a
standard reference when considering which AEAD algorithm to use, and how to use
it.

# Requirements Notation

{::boilerplate bcp14}

# Notation

This document defines limitations in part using the quantities below.

| Symbol  | Description |
|-:-|:-|
| n | Size of the AEAD block cipher (in bits) |
| t | Size of the authentication tag (in bits) |
| l | Length of each message (in blocks)
| s | Total plaintext length (in blocks) |
| q | Number of encryption attempts |
| v | Number of forgery attempts |
| p | Adversary attack probability |

For each AEAD algorithm, we define the confidentiality and integrity advantage
roughly as the advantage an attacker has in breaking the corresponding security
property for the algorithm. Specifically:

- Confidentiality advantage (CA): The advantage of an attacker succeeding in breaking
the confidentiality properties of the AEAD. In this document, the definition of
confidentiality advantage is the increase in the probability that an attacker is
able to successfully distinguish an AEAD ciphertext from the output of an ideal
pseudorandom permutation (PRP).

- Integrity advantage (IA): The probability of an attacker succeeding in breaking
the integrity properties of the AEAD. In this document, the definition of
integrity advantage is the probability that an attacker is able to forge a
ciphertext that will be accepted as valid.

Each application requires a different application of limits in order to keep CA
and IA sufficiently small.  For instance, TLS aims to keep CA below 2^-60 and IA
below 2^-57.

# Calculating Limits

Once an upper bound on CA and IA are determined, this document
defines a process for determining two overall limits:

- Confidentiality limit (CL): The number of bytes of plaintext and maybe
  authenticated additional data (AAD) an application can encrypt before given
  the adversary a non-negligible CA.

- Integrity limit (IL): The number of bytes of ciphertext and maybe auehtnciated
  additional data (AAD) an application can process, either successfully or not,
  before giving the adversary a non-negligible IA.

For an AEAD based on a block function, it is common for these limits to be
expressed instead in terms of the number of blocks rather than bytes.
Furthermore, it might be more appropriate to track the number of messages rather
than track bytes.  Therefore, the guidance is usually based on the total number
of blocks processed (s).  To aid in calculating limits for message-based
protocols, a formulation of limits that includes a maximum message size (l) is
included.

All limits are based on the total number of messages, either the number of
protected messages (q) or the number of forgery attempts (v); which correspond
to CL and IL respectively.

Limits are then derived from those bounds using a target attacker probability.
For example, given an advantage of q\*v and attacker success probability of p,
the algorithm remains secure with respect provided that q\*v <= p. In turn, this
implies that v <= p/q is the corresponding limit.

<!-- I'm not happy with this example as it implies that multiplying q and v is a
     thing, but it is not as far as I know.  Maybe pick a semi-real example. -->

<!-- So p is either CA or IA here.  Do we need to use two characters instead (c
     and i for confidentiality and integrity? e and d for encryption and
     decryption?  s and o for seal and open?) -->

<!-- We have a lot of cases here where it might be nice to express numbers
     differently.  For instance, most of these equations are a lot simpler if
     you use 2^v rather than v because you can say that q = 106-2l-p/2 or
     something like that.  That might help as you can say that CL = 2^q, rather
     than have to awkwardly say that CL is q. -->


# AEAD Limits and Requirements {#limits}

This section summarizes the confidentiality and integrity bounds and limits for modern AEAD algorithms
used in IETF protocols, including: AEAD_AES_128_GCM {{!RFC5116}}, AEAD_AES_256_GCM {{!RFC5116}},
AEAD_AES_128_CCM {{!RFC5116}}, AEAD_CHACHA20_POLY1305 {{!RFC7539}}, AEAD_AES_128_CCM_8 {{!RFC6655}}.

The CL and IL values bound the total number of encryption and forgery queries (q and v).
Alongside each value, we also specify these bounds.

## AEAD_AES_128_GCM and AEAD_AES_256_GCM

The CL and IL values for AES-GCM are derived in {{AEBounds}} and summarized below.

<!-- Do we want to use n and t in formulae, but start by saying that n=t=128? -->

<!-- It seems like the analysis in AEBounds didn't really consider the role that
     the AAD plays in all of this.  What do we want to say about the simplifying
     assumptions here? -->

### Confidentiality Limit

~~~
CA = ((s + q + 1)^2) / 2^127
~~~

This implies the following limit:

~~~
q <= p^(1/2) * 2^(127/2) - s - 1
~~~

### Integrity Limit

~~~
IA = 2 * (v * (l + 1)) / 2^128
~~~

This implies the following limit:

~~~
v <= (p * 2^127) / (l + 1)
~~~
<!-- Let's simplify that `+1` away, it's awkward.  We'll have to clearly signal
     that for l < ? then you might instead want p * 2^(127-2l) instead though. -->

## AEAD_CHACHA20_POLY1305

The only known analysis for AEAD_CHACHA20_POLY1305 {{ChaCha20Poly1305Bounds}}
combines the confidentiality and integrity limits into a single expression,
covered below:
<!-- I've got to say that this is a pretty unsatisfactory situation. -->

~~~
v * (8l / 2^106)
~~~

This advantage is a tight reduction based on the underlying Poly1305 PRF {{Poly1305}}.
It implies the following limit:

~~~
v <= (p * 2^106) / 8l
~~~

## AEAD_AES_128_CCM

The CL and IL values for AEAD_AES_128_CCM are derived from {{?CCM-ANALYSIS=DOI.10.1007/3-540-36492-7_7}}
and specified in the QUIC-TLS mapping specification {{?I-D.ietf-quic-tls}}.

### Confidentiality Limit

~~~
CA: (2l * q)^2 / 2^128
~~~

This implies the following limit:

~~~
q <= sqrt((p * (2^127)) / l^2)
~~~

### Integrity Limit

~~~
IA: v / 2^t + (2l * (v + q))^2 / 2^n
~~~

This implies the following limit:

~~~
TODO(caw)
~~~

## AEAD_AES_128_CCM_8

TODO

# Security Considerations {#sec-considerations}

Many of the formulae in this document depend on simplifying assumptions that are
not universally applicable.  When using this document to set limits, it is
necessary to validate all these assumptions for the setting in which the limits
might apply.  In most cases, the goal is to use assumptions that result in
setting a more conservative limit, but this is not always the case.

# IANA Considerations

This document does not make any request of IANA.

--- back