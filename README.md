# UCAN Invocation Specification v0.1.0

## Editors

* [Brooklyn Zelenka](https://github.com/expede/), [Fission](https://fission.codes/)

## Authors

* [Brooklyn Zelenka](https://github.com/expede/), [Fission](https://fission.codes/)
* [Irakli Gozalishvili](https://github.com/Gozala), [DAG House](https://dag.house/)

## Depends On

* [`ucan-ipld`](https://github.com/ucan-wg/ucan-ipld/)

# 0 Abstract

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# 1 Introduction

## Separation

Why separate the UCAN from the invocation format? Surely the UCAN itself already contains all the infomation required.

I would argue that this is not the case for a few reasons.

1. Authority is not intent to execute. Authority transfer is first-class in UCAN, other actions are not
2. Mixing the two levels means that invocation info will live with the token (as a proof) on subdelegations
3. Other detail, such as scheduling

A job request may be picked up by anyone, PRIOR to UCAN delegation. A handshake or matchmaking may need to be performed.

## Authority Is Not Intention

Or to put it another way: "just because you can, doens't mean you should". Granting a UCAN to a peer means that they are allowed to perform some actions for a period of time. This is a feature not a bug, but also says nothing about the intention of when it should be run. I may grant a collaborative process the ability to perform actions on an ongoing basis (hmm, but vioating POLA)

I could preload the service with `n` UCANs with narrow time windows and `nbf`s in the future. That would certainly be very secure, but it would be less convenient since I need to come online to issue more or to 

# 2 Envelope

UCAN uses a capabilities model. The 

* UCANTO
* zcap
* IPVM

 ``` json
{
  "ucan/invoke": [ "bafyLeft", "bafyRight", "bafyEnd" ]
  "version": "0.1.0",
  /* "nonce": "abcdef" -- I think the nonce inside each UCAN is sufficent? But also can never hurt */
  "siganture": 0xCOFFEE
}
```

# 3 Receipt & Attestation

An invocation receipt is a claim about what the output of an invocation is. A receipt MUST be attested via signature of the principal (the audience of the associated invocation).

Note that this does not guarantee correctness of the result! The level of this statement's veracity MUST be ony taken that the signer claims this to be a fact.

The receipt MUST be signed with by the `aud` from the UCAN.

## 3.1 IPLD

```ipldsch
type Receipt struct {
  i {&UCAN : {URI : Any}} <!-- not sure if this actually works? My guess is that the link doesn't work because it should not  -->
  v SemVer
  n String
  s Bytes
}

type SemVer struct {
  ma Integer
  mi Integer
  pa Integer
  ta optional nullable String
}

type URI = String
```

## 3.2 JSON Example

``` json
{
  "ucan/invocation/receipt": {
    "bafyLeft": {
      "a": 42,
      "example.com": {
        "msg/read": [
          "from": "alice@example.com",
          "text": "hello world"
        ]
      }
    },
    "bafyRight": {
      "sub.example.com?TYPE=TXT": {
        "crud/update": {
          "12345": {
            "http": { 
              "status": 200
            },
            "value": "lorem ipsum"
          }
        }
      }
    }
  },
  "version": "0.1.0",
  "nonce": "xyz",
  "signature": 0xB00
}
```

# 4 Pipelining

> Machines grow faster and memories grow larger. But the speed of light is constant and New York is not getting any closer to Tokyo. As hardware continues to improve, the latency barrier between distant machines will increasingly dominate the performance of distributed computation. When distributed computational steps require unnecessary round trips, compositions of these steps can cause unnecessary cascading sequences of round trips
>
> Robust Composition, Mark Miller

At time of creation, a UCAN MAY not know the concrete value required to scope the resource down sufficiently. This MAY be caused either by invoking them both in the same payload, or following one after another by CID reference.

Variables relative to the result of some other action MAY be used. In this case, the attested (signed) receipt of the previous action MUST be included in the following form:

Refeernced by invocation CID
