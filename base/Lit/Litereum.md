Litereum: a minimal decentralized computer
==========================================

In 2013, the first smart-contract blockchain, Ethereum, was proposed, extending
Bitcoin with a stateful scripting language that allowed arbitrary financial
transactions to be settled without third parties. Notably, Ethereum's built-in
virtual machine made it a general-purpose computer, even though most of the
protocol's complexity wasn't required to achieve Turing completeness. Litereum
is a massive simplification of this concept, trading features for simplicity.
Without a native token, it, technically, isn't a cryptocurrency, even though
tokens can be created internally as stateful programs. In essence, Litereum is
just the Lambda Calculus running in a peer-to-peer network, making it a
politically neutral decentralized computer.

Implementation
--------------

Litereum's design is a combination of 3 components:

1. LitCons: a minimal, proof of work consensus algorithm.

2. LitSign: a minimal, quantum-resistant, hash-based signature scheme.

3. LitCore: a minimal, statically typed affine calculus used for scripting. 

### LitCore

LitCore is a affine, simply typed, functional language featuring recurive
functions, algebraic datatypes, pattern-matching and a global state, called
world. It can be seen as the virtual machine running inside Litereum. Every type
and algorithm required to implement LitCore will be specified below.

#### Bits

A string that can be either empty or any sequence of bits

#### World

The global state of LitCore. It is just a map from names to entries.

#### Entry

An entry on the global state. It can be either an User, a Type, or a Function.

#### User

An external user capable of running authenticated scripts. It has 2 fields:

- `name : Bits`: the name of the user

- `pkey : Bits`: the public key of the user

#### Type

An algebraic datatype. It has two fields:

- `name : Bits`: the name of the type

- `ctors : List<Constructor>`: a list of Constructors

#### Constructor

A Constructor has two fields:

- `name : Bits`: the name of the constructor

- `fields : List<Field>`: a list of Fields

#### Field

A Field has two fields:

- `name : Bits`: the name of the field 

- `type : List<Type>`: the type of the field

The type of a field can be the name of any type defined previously, or a
recursive occurrence of the type being defined.

#### Function

A global, affine, simply typed function that can be called by external users, or
by other functions. Functions are pure, except for one stateful operation that
can rebind other functions in the global state.

A function has 6 fields:

- `name`: the name of the function being defined

- `ownr`: a list of owners that can redefine this function

- `main`: a Term, representing the body of the function

- `iarg`: a list of argument names

- `ityp`: a list of argument types

- `otyp`: the output type of the function

#### Command

A top-level command that alters the global state. Commands are grouped in pages.
Tracing a parallel to conventional blockchains, a command is like a transaction,
and a page is like a block. The main difference is that commands don't need to
be signed. There are 4 variants of commands:

##### 1. `new_type(type)`

Defines a new Type on the global state.

If the type's name already exists, then this command fails.

##### 2. `new_func(func)`

Defines a new Function on the global state.

If the function's name already exists, then this command fails.

If the function's body is ill-typed with a type context initialized with the
function's argument list, then this command fails. The function body
is ill-typed if it doesn't pass the type-check procedure, with a type context
initialized with each variable on the function's argument list.

If the function's body is invalid, then this command fails. The function body is
invalid if it doesn't pass the validate procecure.

##### 3. `new_user(user)`

Defines a new User on the global state.

If the user's name already exists, then this command fails.

##### 4. `with_user(user, signature, term)`

Executes an expression in behalf of an external user.

If the signature is invalid, the command is executed anonymously, with the
username being set as the empty bitstring.

If the expression's body is ill-typed, then this command fails. The function body
is ill-typed if it doesn't pass the type-check procedure with an empty context.

If the expression's body is invalid, then this command fails. The expression
body is invalid if it doesn't pass the validate procedure.

#### Term

A Litereum term is an expression that performs a computation and returns a
value. There are 5 variants:

##### 1. `var(name)`

Represents a variable bound by a global function, or by a pattern-matching
clause. When a function is called, or a pattern-match takes place, the bound
variable will be substituted by its called, or matched, value. Variables must be
affine, which means the same bound variable can only be used at most once, and
must be unique, which means the same name can't be bound to two different
variables, globally.

##### 2. `create(type, form, vals)`

Instantiates a constructor of an algebraic datatype. It has 3 fields:

- `type : Bits`: the name of the type to be created.

- `ctor : Nat`: the index of the constructor to be created.

- `vals : List<Term>`: the values of the fields to be created.

A create variant is well-typed if:

1. The `type` exists on the global state

2. The `ctor` index is smaller than number of constructors in `type`

3. For each index `0 < i < length(vals)`, the value `vals[i]` is well-typed, and
   its type matches the type stored on `fields[i]`, where `fields` is the list
   of fields stored on the constructor of number `ctor` in the selected type

The create variant doesn't compute. It is irreducible.

##### 3. `match`

Performs a pattern-match. It has 4 fields:

- `type : Bits`: the name of the type of the matched value.

- `name : Bits`: a name for the matched value.

- `expr : Term`: the matched value.

- `cses : List<Case>`: a list of cases.

The `match` variant is well-typed if: 

<TODO>

The `match` variant computes if the matched constructor, `expr`, is of the
`create` variant. In this case, the expression reduces to the selected case,
`cses[ctor]`, where `ctor` is the index stored on `expr`, with each variable
bound by the selected case is substituted by the respective field of the matched
constructor. <TODO: improve this paragraph>

##### 4. `call`

Calls an external function, returning a value. It has 4 fields:

- `name : Bits`: the name of the returned value.

- `func : Bits`: the name of the function to be called.

- `args : List<Term>`: the list of arguments.

- `cont : Term`: the body of the call.

The `call` variant is well-typed if: <TODO>

The `call` variant computes: <TODO>

##### 5. `bind`

Binds a global definition, possibly mutating an existing value. It has 3 fields:

- `name : Bits`: the name of the definition to be altered.

- `main : Bits`: the value to be bound.

- `cont : Term`: the body of the bind.

The `bind` variant is well-typed if: <TODO>

The `bind` variant computes: <TODO>

#### Case

A case in a pattern-match. It has 2 fields:

- `flds : List<Bits>`: a list of field names

- `body : Term`: the term returned by that case

LitSign
-------

LitCons uses a configuration of WOTS for message authentication, based on the
Keccak256 function. To be able to sign messages, an user must first generate a
random 256-bit word as its seed. From this seed, the user can generate private
keys by concatenating and hashing `keccak(seed | n | i)` for each natural number
`i` up to `32` (exclusive), where `n` is the number of the private key. That is:

```
function private_key(seed, n):
  private_key = []
  for i from 0 to 32:
    private_key.push(keccak(seed | n | i))
  return private_key
```

A public key can then be generated by taking `keccak^256(w)`, for each word `w`
in the private key, concatenating these results, and taking its hash. That is:

```
function public_key(private_key):
  public_key = []
  for i from 0 to 32:
    hash = private_key[i]
    for j from 0 to 256:
      hash = keccak(hash)
    public_key.push(hash)
  return keccak(public_key)
```

The user must then broadcast his public key as his public identifier, and keep
his private key secret. To sign a message `M` the user must first generate a
256-bit summary of `M`. That summary consists of the last 30 bytes of the
Keccak of the message, prefixed with the sum of these bytes. That is:

```
function summary(message):
  hash    = keccak(message)[2..32]  # last 30 bytes of the hash
  byte_0  = sum(hash) / 256         # first byte of the checksum
  byte_1  = sum(hash) % 256         # second byte of the checksum
  summary = byte_0 | byte_1 | hash  # the 32-byte summary
  return summary
```

Then, for each natural number `i`, up to `32`, the signer must take
`256 - summary[i]` consecutive hashes of `private_key[i]`. The concatenation of
these 32 hashes is the 1024-bytes signature. That is:

```
function sign(private_key, summary):
  signature = []
  for i from 0 to 32:
    value = private_key[i]
    for i from 0 to 256 - summary[i]:
      value = keccak(value)
    signature.push(value)
  return signature
```

A public key can be recovered from a signature and a summary by, for each
natural number `i`, up to `32`, taking `summary[i]` consecutive hashes of
`signature[i]`. That is:

```
function recover(summary, signature):
  public_key = []
  for i from 0 to 32:
    hash = signature[i]
    for i from 0 to summary[i]:
      hash = keccak(hash)
    public_key.push(hash)
  return keccak(public_key)
```

To check if a signature is valid, the verifier must check if the public key
returned by `recover()` matches with the one published. That is:

```
function valid(message, signature, public_key):
  return recover(summary(message), signature) == public_key
```

This signature scheme has 120 bits of classical security, and 80 bits of
post-quantum security. It has a signature size of 1024 bytes, and requires an
average of 4096 hashes per signature and verification. This is an one-time
signature scheme, which means that, once a message is signed with a public key,
that public key must be thrown away, and a new one must be generated. In
Litereum, every time an user makes a transaction, they broadcasts the new
public key to the network.

LitCons
-------

LitCons is a data-only blockchain. It could be described as "Bitcoin without the
coin". It is the consensus layer used by Litereum to order blocks (pages) in its
decentralized network, but it is fully independent from it. Like most components
of Litereum, LitCons's design is extremely minimal. 

A LitCons page is similar to a Bitcoin block, but it has only 4 fields: a
Keccak-256 hash pointing to the previous page, a 256-bit nonce (that can also be
used to store extra data), the miner's address, and a 1280-bytes body that can
store arbitrary data. In Kind, it is represented by the following type:

```
type Lit.Cons.Page {
  new(
    prev: U256            // previous page (32 bytes)
    user: U256            // miner address or identifier
    nonc: U256            // nonce + extra data (32 bytes)
    body: Vector<U256,40> // page contents (1280 bytes)
  )
}
```

Unlike Bitcoin blocks, a LitCons page doesn't hold a set of transactions.
Instead, it stores an arbitrary blob of 1280 bytes of data. The size was chosen
to allow a full block to fit in a small UDP packet, allowing for fast
propagation. There are no "block headers": a page stores all its data. There
aren't native tokens, nor transaction fees. The incentives to include Litereum
commands (transactions) in a page are determined by miners and users: fees can
be paid with any internal token.

Nakamoto Consensus [BITCOIN_WHITEPAPER] is used to publish, propagate and order
LitCons pages in a manner such that network participants agree to a single
canonical list of posts, except a single Keccak-256 is used to take the block
hash and (TODO: parameters here).

Roadmap
-------

Litereum's roadmap has two phases.

2021-2022: Prelude

In this phase, which starts as soon as this paper is released, an initial
prototype of a full Litereum node will be released, including all 3 major
components, LitCons, LitSign and LitCore. The protocol will immediately start
operating in initial test mode, and may require eventual updates and adjustments
to polish it into its permanent form. During this phase, the network will have a
centralized sequencer, which will be responsible for, and only for, ordering the
transactions in chronological order. Everything else, including Proof of Work
(PoW) mining, will remain the same. When the network moves to the next phase, it
will inherit the transaction history and accumulated PoW.

2022-forever: Chronicle

In this phase, which starts when the network has been stable for enough time,
the centralized sequencer will be turned off, and nodes around the world will
validate transactions via Nakamoto consensus, and the protocol will be
considered final and should never require upgrades. This means that, from this
moment on, there will be no core developers with any amount of privileged
political power. Of course, due to the very nature of blockchains, anyone can
implement changes and propose a fork to the network, but we advise users to be
naturally critical and resistant to such improvement proposals, even if they
come from ourselves. If the codebase becomes large and complex enough, it will,
once again, concentrate the power in the hands of a few core developers, which
goes against the proposal.