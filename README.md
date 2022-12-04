# UCAN Closure Specification v0.1.0

## Editors

* [Brooklyn Zelenka](https://github.com/expede/), [Fission](https://fission.codes/)

## Authors

* [Brooklyn Zelenka](https://github.com/expede/), [Fission](https://fission.codes/)
* [Irakli Gozalishvili](https://github.com/Gozala), [DAG House](https://dag.house/)

## Depends On

* [DAG-CBOR](https://ipld.io/specs/codecs/dag-cbor/spec/)
* [UCAN](https://github.com/ucan-wg/spec/)
* [UCAN-IPLD](https://github.com/ucan-wg/ucan-ipld/)
* [Varsig](https://github.com/ChainAgnostic/varsig/)

# 0 Abstract

UCAN Closure defines a format for expressing the intention to run delegated capabilities from a UCAN, and how to promise pipeline invocations.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# 1 Introduction

> Just because you can doesn't mean that you should
>
> — Anonymous

UCAN is a chained-capability format. A UCAN contains all of the information that one would need to perform some task, and the provable authority to do so. This begs the question: can UCAN be used directly as an RPC language?

Some teams have had success with UCAN directly for RPC when the intention is clear from context. This can be successful when there is more information on the channel than the UCAN itself (such as an HTTP path that a UCAN is sent to). However, capability invocation contains strictly more information than delegation: all of the authority of UCAN, plus the command to perform the task.

## 1.1 Intuition

## 1.1.1 Car Keys

Consider the following fictitious scenario:

Akiko is going away for the weekend. Her good friend Boris is going to borrow her car while she's away. They meet at a nearby cafe, and Akiko hands Boris her car keys. Boris now has the capability to drive Alice's car whenever he wants to. Depending on their plans for the rest of the day, Akiko may find Boris quite rude if he immediately leaves the cafe to go for a drive. On the other hand, if Akiko asks Boris to run some last minute pre-vacation errands for that require a car, she may expect Boris to immediately drive off.

## 1.1.2 Lazy vs Eager Evaluation

In a referentially transparent setting, the description of a task is equivalent to having done so: a function and its results are interchangeable. [Programming languages with call-by-need semantics](https://en.wikipedia.org/wiki/Haskell) have shown that this can be an elegant programming model, especially for pure functions. However, _when_ something will run can sometimes be unclear. 

Most languages use eager evaluation. Eager languages must contend directly with the distinction between a reference to a function and a command to run it. For instance, in JavaScript, adding parentheses to a function will run it. Omitting them lets the program pass around a reference to the function without immediately invoking it.

``` js
const message = () => alert("hello world")
message // Nothing happens
message() // A message interups the user
```

Delegating a capability is like the statement `message`. Closure is akin to `message()`. It's true that sometimes we know to run things from their surrounding context without the parentheses:

```js
[1,2,3].map(message) // Message runs 3 times 
```

However, there is clearly a distinction between passing a function and invoking it. The same is true for capabilities: delegating the authority to do something is not the same as asking for it to be done immediately, even if sometimes it's clear from context.

## 1.2 Separation of Concerns

Information about the scheduling, order, and pipelining of tasks is orthogonal to the flow of authority. An agent collaborating with the original executor does not need to know that their call is 3 invocations deep; they only need to know that they been asked to perform some task by the latest invoker.

As we shall see in the [discussion of promise pipelining](#6-promise-pipelining), asking an agent to perform a sequence of tasks before you know the exact parameters requires delegating capabilities for all possible steps in the pipeline. Pulling pipelining detail out of the core UCAN spec serves two functions: it keeps the UCAN spec focused on the flow of authority, and makes salient the level of de facto authority that the executor has (since they can claim any value as having returned for any step).

```
  ────────────────────────────────────────────Time──────────────────────────────────────────────────────►

┌──────────────────────────────────────────Delegation─────────────────────────────────────────────────────┐
│                                                                                                         │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐         ┌─────────┐                ┌─────────┐                 │
│  │         │   │         │   │         │         │         │                │         │                 │
│  │  Alice  ├──►│   Bob   ├──►│  Carol  ├────────►│   Dan   ├───────────────►│  Erin   │                 │
│  │         │   │         │   │         │         │         │                │         │                 │
│  └─────────┘   └─────────┘   └─────────┘         └─────────┘                └─────────┘                 │
│                                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────Invocation─────────────────────────────────────────────────────┐
│                                                                                                         │
│                              ┌─────────┐         ┌─────────┐                                            │
│                              │         │         │         │                                            │
│                              │  Carol  ╞═══All══►│   Dan   │                                            │
│                              │         │         │         │                                            │
│                              └─────────┘         └─────────┘                                            │
│                                                                                                         │
│                                                  ┌─────────┐                              ┌─────────┐   │
│                                                  │         │                              │         │   │
│                                                  │   Dan   ╞═══════════Update DB═════════►│  Erin   │   │
│                                                  │         │                              │         │   │
│                                                  └─────────┘                              └─────────┘   │
│                                                                                                         │
│                                                           ┌─────────┐                ┌─────────┐        │
│                                                           │         │                │         │        │
│                                                           │   Dan   ╞═══Read Email══►│  Erin   │        │
│                                                           │         │           ▲    │         │        │
│                                                           └─────────┘           ┆    └─────────┘        │
│                                                                               With                      │
│                                                                               Result                    │
│                                                                  ┌─────────┐   Of         ┌─────────┐   │
│                                                                  │         │    ┆         │         │   │
│                                                                  │   Dan   ╞════Set DNS══►│  Erin   │   │
│                                                                  │         │              │         │   │
│                                                                  └─────────┘              └─────────┘   │
│                                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## 1.3 A Note On Serialization

The JSON examples below are given in [DAG-JSON](https://ipld.io/docs/codecs/known/dag-json/), but UCAN Closure is actually defined as IPLD. This makes UCAN Closure agnostic to encoding. DAG-JSON follows particular conventions around wrapping CIDs and binary data in tags like so:

``` json
// CID
{"/": "Qmf412jQZiuVUtdgnB36FXFX7xg5V6KEbSJ4dpQuhkLyfD"}

// Bytes (e.g. a signature)
{"/": {"bytes": "s0m3Byte5"}}
```

This format help disambiguate type information in generic DAG-JSON tooling. However, your presentation need not be in this specific format, as long as it can be converted to and from this cleanly. As it is used for the signature format, DAG-CBOR is RECOMMENDED.

## 1.4 Signatures

All payloads described below MUST be signed with a [Varsig](https://github.com/ChainAgnostic/varsig/).

# 2 High-Level Concepts

## 2.1 Roles

Closure adds two new roles to UCAN: invoker and executor. The existing UCAN delegator and delegate principals MUST persist to the invocation.

| UCAN Field | Delegation                             | Closure                      |
|------------|----------------------------------------|---------------------------------|
| `iss`      | Delegator: transfer authority (active) | Invoker: request task (active)  |
| `aud`      | Delegate: gain authority (passive)     | Executor: perform task (active) |

### 2.1.1 Invoker

The invoker signals to the executor that a task associated with a UCAN SHOULD be performed.

The invoker MUST be the UCAN delegator. Their DID MUST be authenticated in the `iss` field of the contained UCAN.

### 2.1.2 Executor

The executor is directed to perform some task described in the UCAN by the invoker.

The executor MUST be the UCAN delegate. Their DID MUST be set the in `aud` field of the contained UCAN.

## 2.2 Components 

![](./diagrams/concepts.svg)

### 2.2.1 Closure

A [closure](#3-closure) is like a deferred function application: a request to perform some action on a resource with specific inputs.

### 2.2.2 Receipt

A [receipt](FIXME) describes the output of an invocation, referenced by its pointer.

### 2.2.3 Promise

A [promise](FIXME) is a reference to the receipt of an action that has yet to return a receipt.

### 2.2.4 Batch

A [batch](FIXME) is a way of requesting more than one action at once.

### 2.2.5 Pipeline

A [pipeline](FIXME) is a batch that includes promises. This allows for the automatic chaining of actions based on their outputs.

### 2.2.6 Memoization

A [distributed memoization table](FIXME) (DMT) is a way of sharing receipts in a consistent lookup table.

## 2.3 IPLD Schema

``` ipldsch
type Closure struct {
  with   URI
  do     Ability
  inputs Any
}

type Task struct {
  inv  &Closure
  meta {String : Any} (implicit {})
}

type Batch union {
  | Set   [Task]
  | Named {String : Task}
}

type Invocation struct {
  run  Batch
  uiv  SemVer
  prf  [&UCAN]
  nnc  String
  meta {String : Any} (implicit {})
  sig  Varsig
}

type TaskPtr struct {
  inv InvPtr
  lbl optional String
} representation tuple

type InvPtr union {
  | "/" -- "Local": Relative to the current invocation
  | &Invocation
}

type Status enum {
  | Ok  ("ok")
  | Err ("err")
}

type Promise struct {
  ipr InvPtr
  lbl String
  sts Status (implicit "ok")
} 

type Receipt struct {
  ran  Pointer
  out  {String : Result}
  meta {String : Any}
  sig  Varsig
} 

type Result union {
  | Success
  | Failure
}

type Failure struct {
  err   nullable String
  trace Any
}

type Success struct {
  val Any
  rec optional &SignedReceipt
}
```

# 3 Closure

An invocation is the smallest unit of work that can be requested from a UCAN. It invokes one specific `(resource, ability, inputs)` triple. The inputs are freeform, and depend on the specific resource and ability being interacted with.

Closures ........... FIXME

Using the JavaScript analogy from the introduction, this is similar to wrapping a call in a nullary function.

``` json
{
  "with": "mailto://alice@example.com",
  "do": "msg/send",
  "inputs": {
    "to": ["bob@example.com", "carol@example.com"],
    "subject": "hello",
    "body": "world"
  }
}
```

``` js
() => msg.send("mailto:alice@example.com", {
  to: ["bob@example.com", "carol@example.com"],
  subject: "hello",
  body: "world"
})
```

 
## 3.1 Fields

``` ipldsch
type Closure struct {
  with   URI
  do     Ability
  inputs Any
}
```

### 3.1.1 Resource

The `with` field MUST contain the [URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier) of the resource being accessed. If the resource being accessed is some static data, it is RECOMMENDED to reference it by the [`data`](https://en.wikipedia.org/wiki/Data_URI_scheme), `ipfs`, or [`magnet`]() URI schemes.

### 3.1.2 Ability

The `do` field MUST contain a [UCAN Ability](https://github.com/ucan-wg/spec/#23-ability). This field can be thought of as the message or trait being sent to the resource.

### 3.1.3 Inputs

The `inputs` field MUST contain any arguments expected by the URI/Ability pair. This MAY be different between different URIs and Abilities, and is thus left to the executor to define the shape of this data.

### 3.2 DAG-JSON Examples

Interactig with an HTTP API

``` json
{
  "with": "https://example.com/blog/posts",
  "do": "crud/create",
  "inputs": {
    "headers": {
      "content-type": "application/json"
    },
    "payload": {
      "title": "How UCAN Closures Changed My Life",
      "body": "This is the story of how one spec changed everything...",
      "topics": ["authz", "journal"],
      "draft": true
    }
  }
}
```

Sending Email

``` json
{
  "with": "mailto:akiko@example.com",
  "do": "msg/send",
  "inputs": {
    "to": ["boris@example.com", "carol@example.com"],
    "subject": "Coffee",
    "body": "Hey you two, I'd love to get coffee sometime and talk about UCAN Closures!"
  }
}
```

Running WebAssembly

``` json
{
  "with": "data:application/wasm;base64,AHdhc21lci11bml2ZXJzYWwAAAAAAOAEAAAAAAAAAAD9e7+p/QMAkSAEABH9e8GowANf1uz///8UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAP////8AAAAACAAAACoAAAAIAAAABAAAACsAAAAMAAAACAAAANz///8AAAAA1P///wMAAAAlAAAALAAAAAAAAAAUAAAA/Xu/qf0DAJHzDx/44wMBqvMDAqphAkC5YAA/1mACALnzB0H4/XvBqMADX9bU////LAAAAAAAAAAAAAAAAAAAAAAAAAAvVXNlcnMvZXhwZWRlL0Rlc2t0b3AvdGVzdC53YXQAAGFkZF9vbmUHAAAAAAAAAAAAAAAAYWRkX29uZV9mAAAADAAAAAAAAAABAAAAAAAAAAkAAADk////AAAAAPz///8BAAAA9f///wEAAAAAAAAAAQAAAB4AAACM////pP///wAAAACc////AQAAAAAAAAAAAAAAnP///wAAAAAAAAAAlP7//wAAAACM/v//iP///wAAAAABAAAAiP///6D///8BAAAAqP///wEAAACk////AAAAAJz///8AAAAAlP///wAAAACM////AAAAAIT///8AAAAAAAAAAAAAAAAAAAAAAAAAAET+//8BAAAAWP7//wEAAABY/v//AQAAAID+//8BAAAAxP7//wEAAADU/v//AAAAAMz+//8AAAAAxP7//wAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAU////pP///wAAAAAAAQEBAQAAAAAAAACQ////AAAAAIj///8AAAAAAAAAAAAAAADQAQAAAAAAAA==",
  "do": "executable/run",
  "inputs": {
    "func": "add_one",
    "args": [42]
  }
}
```

# 4 Task

A Task is an 

FIXME wrappr vs subtyping?

``` ipldsch
type Task struct {
  inv  &Closure
  meta {String : Any} (implicit {})
}
```

# 5 Batch

In many situations, sending multiple requests in a batch is more efficient. A Batch is a collection of Closures, either as an array or a named map.

``` ipldsch
type Batch union {
  | Set   [Task]
  | Named {String : Task}
}
```

## 5.1 Fields

Each Task in a Batch contains a reference to [Closure](FIXME) itself, plus optional metadata.


``` json
{
  "batch": {
    "left": {
      
    },
    "middle": {
    
    },
    "right": {
    
    }
  }
}
```

# 6 Invocation

Note that there is are no signature or UCAN proof fields in the Closure struct. To allow for better nesting inside of other formats, these fields are broken out into an envelope for when Closures are used standalone:

An Invocation authorizes one or more Closures to be run. There are a few invariants that MUST hold between the `run`, `prf` and `sig` fields:

* All of the `run` Closures MUST be provably authorized by the UCANs in the `prf` field
* All of the `prf` UCANs MUST list the Executor in their `aud` field
* All of the `prf` UCANs MUST list the Invoker in their `iss`
* The `sig` field MUST be produced by the Invoker

## 6.1 IPLD Schema

``` ipldsch
type ClosureInvocation struct {
  uiv  SemVer
  run  Batch
  prf  [&UCAN]
  nnc  String
  meta {String : Any} (implicit {})
  sig  Varsig
}
```

## 6.2 Fields

### 6.2.1 UCAN Closure Version

The `uiv` field MUST contain the Semver-frmatted version of the UCAN Closure Specification that this struct conforms to.

### 6.2.2 Closure

The `run` field MUST contain a link to the [Closure](#31-single-invocation) itself.

### 6.2.3 Proofs

The `prf` field MUST contain links to any UCANs that provide the authority to run the actions. All of the outermost `aud` fields MUST be set to the [Executor](#212-executor)'s DID. All of the outermost `iss` field MUST be set to the [Invoker](#211-invoker)'s DID.

### 6.2.4 Nonce

The `nnc` field MUST include a random nonce field expressed in ASCII. This field ensures that multiple invocations are unique.

### 6.2.5 Metadata

If present, the OPTIONAL `meta` map MAY contain freeform fields. This provides a place for extension of the invocation type.

Data inside the `meta` field SHOULD NOT be used for [memoization]() and [receipts]().

### 6.2.6 Signature

The `sig` field MUST contain a [Varsig](https://github.com/ChainAgnostic/varsig) of the `inv`, `prf`, and `nnc` fields.

## 6.3 DAG-JSON Examples

``` json
{
  "uiv": "0.1.0"
  "run": {"/": "bafkreidu4n2252jl3zjhbhcpnxtau5zcy5f6lipgqcik6b3o2jkkjt5ali"},
  "prf": [
    {"/": "bafkreie2cyfsaqv5jjy2gadr7mmupmearkvcg7llybfdd7b6fvzzmhazuy"},
    {"/": "bafkreibbz5pksvfjyima4x4mduqpmvql2l4gh5afaj4ktmw6rwompxynx4"}
  ],
  "nnc": "6c*97-3=",
  "meta": {
    "notes/personal": "I felt like making an invocation today!",
    "ipvm/config": {
      "time": [5, "minutes"],
      "gas": 3000
    }
  }
  "sig": {"/": {"bytes:": "5vNn4--uTeGk_vayyPuNTYJ71Yr2nWkc6AkTv1QPWSgetpsu8SHegWoDakPVTdxkWb6nhVKAz6JdpgnjABppC7"}}
}
```

# 7 Invocation Pointer

``` ipldsch
type InvPtr union {
  | "/" -- "Local": Relative to the current invocation
  | &ClosureInvocation
}

type ClosurePointer struct {
  envl  InvPtr
  label optional String
} representation tuple
```

``` json
["/", "some-label"]
[{"/": "bafkreiddwsbb7fasjb6k6jodakprtl4lhw6h3g4k7tew7vwwvd63veviie"}, "some-label"]
[{"/": "bafkreiddwsbb7fasjb6k6jodakprtl4lhw6h3g4k7tew7vwwvd63veviie"}, {"/": {"bytes": "bafkreidlqsd6nh6hdgwhr4machsvusobpvn4qfrxfgl5vowoggzk2xpldq"}}]
```

# 8 Result

``` ipldsch
type Result union {
  | Success
  | Failure
}

type Failure struct {
  err   nullable String
  trace Any
}

type Success struct {
  val Any
  rec optional &SignedReceipt
}
```

# 9 Receipt

``` ipldsch
type Receipt struct {
  ran  Pointer
  out  {String : Result}
  meta {String : Any}
  sig  Varsig
} 
```

An invocation receipt is an attestation of the output of an invocation. A receipt MUST be signed by the Executor (the `aud` of the associated UCANs).

**NB: a Receipt this does not guarantee correctness of the result!** The statement's veracity MUST be only understood as an attestation from the executor.

Receipts don't have their own version field. Receipts MUST use the same version as the invocation that they contain.

## 9.1 Fields


### 9.1.1 Closure

The `inv` field MUST include a link to the Closure that the Receipt is for.

### 9.1.2 Output

The `out` field MUST contain the output of steps of the call graph, indexed by the task name inside the invocation. If the invocation is the implicit `"*"`, then the base64 hash of the concatenation of the URI, Ability and extensional fields MUST be used.

The `out` field MAY omit any tasks that have not yet completed, or results which are not public. An `Closure` may be associated to zero or more `Receipts`.

A `Result` MAY include recursive `Receipt` CIDs in on the `Success` branch. As a Task may require subdelegation, the OPTIONAL `rec` field MAY be used to include recursive `Receipt`s.

### 9.1.3 Metadata Fields

The metadata field MAY be omitted or used to contain additional data about the receipt. This field MAY be used for tags, commentary, trace information, and so on.

### 9.1.4 Signature

The `sig` field MUST contain a [Varsig](https://github.com/ChainAgnostic/varsig) of the `inv`, `out`, and `meta` fields. The signature MUST be generated by the Executor, which means the public key in the `aud` field of the UCANs backing the Closure.

## 9.2 DAG-JSON Examples

``` json
{
  "inv": [{"/": "bafkreifzjut3te2nhyekklss27nh3k72ysco7y32koao5eei66wof36n5e"}],
  "out": {
    "status": "ok",
    "value": [
      {
        "from": "alice@example.com",
        "text": "Hello world!"
      }, 
      {
        "from": "bob@example.com",
        "text": "What's up?"
      }
    ]
  },
  "meta": {
    "time": [400, "hours"],
    "retries": 2
  },
  "sig": {"/": {"bytes": "bdNVZn_uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT-SRK6v_SX8bjt_VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA"}}
}
```

# 10 Promise

``` ipldsch
type Promise struct {
  envelope InvPtr
  label    String
  status   Status (implicit "ok")
} 

type Status enum {
  | Ok  ("ok")
  | Err ("err")
}
```

If there are dependencies or ordering required, then you need a promise pipeline
















    
### 3.1.1 Version

The `v` field MUST contain the version of the invocation object  schema.

### 3.1.2 Proofs

The `prf` field MUST contain CIDs pointing to the UCANs that provide the authority to run these tasks. The elements of this array MUST be sorted in ascending numeric order. Restricting the outmost UCANs to only the authority required for the current invocation is RECOMMENDED.

### 3.1.3 Run Capabilities

The `run` field MUST reference the tasks contained in the UCAN are to be run during the invocation. To run all tasks in the underlying UCAN, the `"*"` value MUST be used. If only specific tasks (or [pipelines](#6-promise-pipelining)) are intended to be run, then they MUST be prefixed with an arbitrary label and treated as a UCAN attenuation: all tasks MUST be backed by a matching capability of equal or greater authority.

#### 3.1.3.1 Promises

The only difference from general capabilities is that [promises](#6-promise-pipelining) MAY also be used as inputs to attenuated fields.

If a capability input has the key `"_"` and the value is a promise, the input MUST be discarded and only used for determining sequencing tasks.

### 3.1.4 Nonce

The `nnc` field MUST contain a unique nonce. This field exists to enable making the CID of each invocation unique. While this field MAY be an empty string, it is NOT RECOMMENDED.

### 3.1.5 Extended Fields

The OPTIONAL `ext` field MAY contain arbitrary data. If not present, the `ext` field MUST be interpreted as `null`, including for [signature](#315-signature).

## 3.2 IPLD Schema

``` ipldsch
type SignedClosure struct {
  inv Closure (rename "ucan/invoke") 
  sig Varsig
}

type Closure struct {
  v   SemVer  -- Version
  prf [&UCAN] -- Authority to run this invocation
  run Scope   -- Which tasks to invoke
  nnc String  -- Nonce
  ext nullable Any (implicit null) -- Extended fields
}

type Scope union {
  | All "*"
  | {String : Task}
}

type Task struct {
  with   URI 
  do     Ability
  inputs Any
  meta   {String : Any}
} 
```

## 3.3 JSON Examples

### 3.3.1 Run All

 ``` js
{
  "ucan/invoke": {
    "v": "0.1.0",
    "prf": [
      {"/": "bafkreie2cyfsaqv5jjy2gadr7mmupmearkvcg7llybfdd7b6fvzzmhazuy"},
      {"/": "bafkreibbz5pksvfjyima4x4mduqpmvql2l4gh5afaj4ktmw6rwompxynx4"}
    ],
    "run": "*"
    "nnc": "abcdef",
    "ext": null
  },
  "sig": {"/": { "bytes:": "bdNVZn_uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT-SRK6v_SX8bjt-VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA" }}
}
```

### 3.3.2 Promise Pipelines

Promise pipelines are handled in more detail in [§6](#6-promise-pipelining). In brief, they enable taking the output of a task that has not yet run as the input to another task. Here is a simple example:

``` js
{
  "ucan/invoke": {
    "v": "0.1.0",
    "nnc": "02468",
    "ext": null,
    "prf": [
      {"/": "bafkreie2cyfsaqv5jjy2gadr7mmupmearkvcg7llybfdd7b6fvzzmhazuy"},
      {"/": "bafkreibbz5pksvfjyima4x4mduqpmvql2l4gh5afaj4ktmw6rwompxynx4"}
    ],
    "run": {
      "notify-bob": {
        "with": "mailto://alice@example.com",
        "do": "msg/send",
        "inputs": [
          {
            "to": "bob@example.com",
            "subject": "DNSLink for example.com",
            "body": "Hello Bob!"
          }
        ]
      },
      "log-as-done": {
        "with": "https://example.com/report"
        "do": "crud/update"
        "inputs": {
          "from": "mailto://alice@exmaple.com",
          "to": ["bob@exmaple.com"],
          "event": "email-notification",
          "value": {"ucan/promise": ["/", "notify-bob"]} // Pipelined promise
        }
      }
    }
  },
  "sig": {"/": {"bytes:": "5vNn4--uTeGk_vayyPuNTYJ71Yr2nWkc6AkTv1QPWSgetpsu8SHegWoDakPVTdxkWb6nhVKAz6JdpgnjABppC7"}}
}
```


# 5 Pointers

# 6 Promise Pipelining

> Machines grow faster and memories grow larger. But the speed of light is constant and New York is not getting any closer to Tokyo. As hardware continues to improve, the latency barrier between distant machines will increasingly dominate the performance of distributed computation. When distributed computational steps require unnecessary round trips, compositions of these steps can cause unnecessary cascading sequences of round trips
>
> — [Mark Miller](https://github.com/erights), [Robust Composition](http://www.erights.org/talks/thesis/markm-thesis.pdf)

At UCAN creation time, the UCAN MAY not yet have all of the information required to construct the next request in a sequence. Waiting for each request to complete before proceeding to the next task has a performance impact due to the amount of latency. [Promise pipelining](http://erights.org/elib/distrib/pipeline.html) is a solution to this problem: by referencing a prior invocation, a pipelined invocation can direct the executor to use the output of earlier invocations into the input of a later one. This liberates the invoker from waiting for each step.

The authority to execute later task often cannot be fully attenuated in advance, since the executor controls the reported output of the prior step in a pipeline. When choosing to use pipelining, the invoker MUST delegate capabilities for any of the possible outputs. If tight control is required to limit authority, pipelining SHOULD NOT be used.

Promises MUST resolve to a [`Result`](#42-ipld-schema). If a promise resolves to the `Success` branch, the value in the `val` MUST be extracted and substituted for the promise. Behavior is left undefined if the promise returns on the `Failure` branch.

Values MUST only be pipelined if they resolve to the `"ok"` branch of the `Result`. In the success case, the value inside the `"ok"` field MUST be extracted and replace the promise.

A promise MAY be placed in any Task field. Substituting into the `with` and `do` fields is NOT RECOMMENDED in fully trustless contexts, as it makes it difficult to understand what is involved in the invocation in advance.

## 6.1 Promises
 
## 6.1.1 Fields

| Field          | Type                | Description                     | Required |
|----------------|---------------------|---------------------------------|----------|
| `ucan/promise` | `ClosurePointer` | The Closure being referenced | Yes      |

The above table MUST be serialized as a tuple. In JSON, this SHOULD be represented as an array containing the values (but not keys) sequenced in the order they appear in the table.

## 6.1.2 IPLD Schema

``` ipldsch
type Promise struct {
  promised TaskPointer (rename "ucan/")
}
```

## 6.1.3 JSON Examples


## 6.2 Pipelining

Pipelining uses promises as inputs to determine the required dataflow graph. The following examples both express the following dataflow graph:

```
              ┌────────────────────────────┐
              │                            │
              │ dns://example.com?TYPE=TXT │
              │        crud/update         │
              │                            │
              └───────┬──────────┬─────────┘
                      │          │
                      │          │
┌─────────────────────▼───┐  ┌───▼────────────────────────┐
│                         │  │                            │
│ mailto:alice@exaple.com │  │ mailto://alice@example.com │
│         msg/send        │  │          msg/send          │
│     bob@example.com     │  │      carol@exmaple.com     │
│                         │  │                            │
└─────────────────────┬───┘  └───┬────────────────────────┘
                      │          │
                      │          │
             ┌────────▼──────────▼────────┐
             │                            │
             │ https://example.com/events │
             │         crud/create        │
             │                            │
             └────────────────────────────┘
```

#### 6.2.1 Batched 

 ``` json
{
  "ucan/invoke": {
    "v": "0.1.0",
    "nnc": "abcdef",
    "prf": [
      {"/": "bafkreie2cyfsaqv5jjy2gadr7mmupmearkvcg7llybfdd7b6fvzzmhazuy"},
      {"/": "bafkreibbz5pksvfjyima4x4mduqpmvql2l4gh5afaj4ktmw6rwompxynx4"}
    ],
    "run": {
      "update-dns" : {
        "with": "dns://example.com?TYPE=TXT",
        "do": "crud/update",
        "inputs": { 
          "value": "hello world",
          "content-type": "text/plain; charset=utf-8"
        }
      },
      "notify-bob": {
        "with": "mailto://alice@example.com",
        "do": "msg/send",
        "inputs": [
          {
            "to": "bob@example.com",
            "subject": "DNSLink for example.com",
            "body": {"ucan/promise": ["/", "update-dns"]}
          }
        ]
      },
      "notify-carol": {
        "with": "mailto://alice@example.com",
        "do": "msg/send",
        "inputs": [
          {
            "to": "carol@example.com",
            "subject": "DNSLink for example.com",
            "body": {"ucan/promise": ["/", "update-dns"]}
          }
        ]
      },
      "log-as-done": {
        "with": "https://example.com/report",
        "do": "crud/update",
        "inputs": [
          {
            "from": "mailto://alice@exmaple.com",
            "to": ["bob@exmaple.com", "carol@example.com"],
            "event": "email-notification",
          },
          {
            "_": {"ucan/promise": ["/", "notify-bob"]}
          },
          {
            "_": {"ucan/promise": ["/", "notify-carol"]}
          }
        ]
      }
    }
  },
  "sig": {"/": {"bytes": "bdNVZn_uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT-SRK6v_SX8bjt-VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA"}}
}
```

### 6.2.2 Serial Pipeline

```
                ┌────────────────────────────┐
                │                            │
                │ dns://example.com?TYPE=TXT │
                │        crud/update         │
                │                            │
                └───────┬──────────┬─────────┘
                        │          │
                        │          │
                        │          │
                        │      ┌───▼────────────────────────┐
                        │      │                            │
                        │      │ mailto://alice@example.com │
                        │      │          msg/send          │
                        │      │      carol@exmaple.com     │
                        │      │                            │
                        │      └───┬────────────────────────┘
                        │          │
┌───────────────────────┼──────────┼──────────┐
│                       │          │          │
│ ┌─────────────────────▼───┐      │          │
│ │                         │      │          │
│ │ mailto:alice@exaple.com │      │          │
│ │         msg/send        │      │          │
│ │     bob@example.com     │      │          │
│ │                         │      │          │
│ └─────────────────────┬───┘      │          │
│                       │          │          │
│                       │          │          │
│              ┌────────▼──────────▼────────┐ │
│              │                            │ │
│              │ https://example.com/events │ │
│              │         crud/create        │ │
│              │                            │ │
│              └────────────────────────────┘ │
│                                             │
└─────────────────────────────────────────────┘
```

 ``` json
{
  "ucan/invoke": {
    "v": "0.1.0",
    "nnc": "abcdef",
    "prf": [{"/": "bafkreifzjut3te2nhyekklss27nh3k72ysco7y32koao5eei66wof36n5e"}],
    "run": {
      "update-dns": {
        "with": "dns://example.com?TYPE=TXT",
        "do": "crud/update",
        "inputs": [
          { 
            "value": "hello world",
            "content-type": "text/plain; charset=utf-8"
          }
        ]
      }
    }
  },
  "sig": {"/": {"bytes": kQHtTruysx4S8SrvSjTwr6ttTLzc7dd7atANUYT-SRK6v_SX8bjHegWoDak2x6vTAZ6CcVKBt6JGpgnjABpsoL"}}
}
 
{
  "ucan/invoke": {
    "v": "0.1.0",
    "nnc": "12345",
    "prf": [
      {"/": "bafkreie2cyfsaqv5jjy2gadr7mmupmearkvcg7llybfdd7b6fvzzmhazuy"},
      {"/": "bafkreibbz5pksvfjyima4x4mduqpmvql2l4gh5afaj4ktmw6rwompxynx4"}
    ],
    "run": {
      "notify-carol": {
        "with": "mailto://alice@example.com",
        "do": "msg/send",
        "inputs": [
          {
            "to": "carol@example.com",
            "subject": "DNSLink for example.com",
            "body": {"ucan/promise": ["bafkreieimb4hvcwizp74vu4xfk34oivbdojzqrbpg2y3vcboqy5hwblmeu", "update-dns"]}
          }
        ]
      }
    }
  },
  "sig": {"/": {"bytes": "XZRSmp5cHaXX6xWzSTxQqC95kQHtTruysx4S8SrvSjTwr6ttTLzc7dd7atANUQJXoWThUiVuCHWdMnQNQJgiJi"}}
}

{
  "ucan/invoke": {
    "v": "0.1.0",
    "nnc": "02468",
    "ext": null,
    "run": {
      "notify-bob": {
        "with": "mailto://alice@example.com",
        "do": "msg/send",
        "inputs": [
          {
            "to": "bob@example.com",
            "subject": "DNSLink for example.com",
            "body": {"ucan/promise": [{"/": "bafkreieimb4hvcwizp74vu4xfk34oivbdojzqrbpg2y3vcboqy5hwblmeu"}, "update-dns"]}
          }
        ]
      },
      "log-as-done": {
        "with": "https://example.com/report",
        "do": "crud/update",
        "inputs": [
          {
            "from": "mailto://alice@exmaple.com",
            "to": ["bob@exmaple.com", "carol@example.com"],
            "event": "email-notification",
          },
          {
            "_": {"ucan/promise": ["/", "notify-bob"]}
          },
          {
            "_": {"ucan/promise": [{"/": "bafkreidcqdxosqave5u5pml3pyikiglozyscgqikvb6foppobtk3hwkjn4"}, "notify-carol"]}
          }
        ]
      }
    }
  },
  "sig": {"/": {"bytes": "5vNn4--uTeGk_vayyPuNTYJ71Yr2nWkc6AkTv1QPWSgetpsu8SHegWoDakPVTdxkWb6nhVKAz6JdpgnjABppC7"}}
}
```

# 7 Appendix

## 7.1 Support Types

``` ipldsch
type CID String
type URI String
type Ability String
type Path String
type Varsig Bytes

type SemVer struct {
  num NumVer
  tag optional String
} representation stringjoin {
  join "+"
}

type NumVer struct {
  ma Integer
  mi Integer
  pa Integer
} representation stringjoin {
  join "."
}
```

# 8 Prior Art

[ucanto RPC](https://github.com/web3-storage/ucanto) from DAG House is a production system that uses UCAN as the basis for an RPC layer.

The [Capability Transport Protocol (CapTP)](http://erights.org/elib/distrib/captp/index.html) is one of the most influential object-capability systems, and forms the basis for much of the rest of the items on this list.

The [Object Capability Network (OCapN)](https://github.com/ocapn/ocapn) protocol extends CapTP with a generalized networking layer. It has implementations from the [Spritely Institute](https://www.spritely.institute/) and [Agoric](https://agoric.com/). At time of writing, it is in the process of being standardized.

[Electronic Rights Transfer Protocol (ERTP)](https://docs.agoric.com/guides/ertp/) builds on top of CapTP for blockchain & digital asset use cases.

[Cap 'n Proto RPC](https://capnproto.org/) is an influential RPC framework [based on concepts from CapTP](https://capnproto.org/rpc.html#specification).

# 9 Acknowledgements

Many thanks to [Mark Miller](https://github.com/erights) for his [pioneering work](http://erights.org/talks/thesis/markm-thesis.pdf) on [capability systems](http://erights.org/).

Many thanks to [Luke Marsen](https://github.com/lukemarsden) and [Simon Worthington](https://github.com/simonwo) for their feedback on invocation model from their work on [Bacalhau](https://www.bacalhau.org/) and [IPVM](https://github.com/ipvm-wg).

Thanks to [Marc-Antoine Parent](https://github.com/maparent) for his discussions of the distinction between declarations and directives both in and out of a UCAN context.

Many thanks to [Quinn Wilton](https://github.com/QuinnWilton) for her discussion of speech acts, the dangers of signing canonicalized data, and ergonomics.

Thanks to [Blaine Cook](https://github.com/blaine) for sharing their experiences with OAuth 1, irreversible design decisions, and advocating for keeping the spec simple-but-evolvable.

Thanks to [Philipp Krüger](https://github.com/matheus23/) for the enthusiastic feedback on the overall design and encoding.

Thanks to [Christine Lemmer-Webber](https://github.com/cwebber) for the many conversations about capability systems and the programming models that they enable.

Thanks to [Rod Vagg](https://github.com/rvagg/) for the clarifications on IPLD Schema implicits and the general IPLD worldview.














FIXME

FIXME invoke an underlying ability from a specific ucan?
