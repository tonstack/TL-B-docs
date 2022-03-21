### TL-B Language

TL-B (Type Language - Binary) serves to describe the type system, constructors, and existing functions. For example we can use TL-B schemes to build binary structures associated with the TON blockchain. Special TL-B parsers can read schemes to deserialize binary data into different objects.

### Table of contents
- [TL-B Language](#tl-b-language)
- [Table of contents](#table-of-contents)
- [Overview](#overview)
- [Constructors](#constructors)
- [Field definitions](#field-definitions)
- [Conditional (optional) fields](#conditional-optional-fields)
- [Namespaces](#namespaces)
- [Comments](#comments)
- [Library usage](#library-usage)
- [IDE Support](#ide-support)
- [Useful sources](#useful-sources)

### Overview

We refer to any set of TL-B constructs as TL-B documents. A TL-B document usually consists of declarations of types (i.e. their constructors) and functional combinators. The declaration of each combinator ends with a semicolon(`;`).

Here is an example of a possible TL-B document.    
<img alt="tlb structure" src="img/tlb.drawio.svg" width="100%">

### Constructors

Constructors are used to specify the type of combinator, including the state at serialization. For example constructors can also be used when you want to specify an `op` in query to a smart contract in the TON.

```c++
// ....
hm_edge#_ {n:#} {X:Type} {l:#} {m:#} label:(HmLabel ~l n)
{n = (~m) + l} node:(HashmapNode m X) = Hashmap n X;

hmn_leaf#_ {X:Type} value:X = HashmapNode 0 X;
// ....
```

The left-hand side of each equation describes a way to define, or even to serialize, a value of the type indicated in the right-hand side. Such a description begins with the name of a constructor, such as `hm_edge` or `hml_long`, immediately followed by an optional constructor tag, such as `#_` or `$10`, which describes the bitstring used to encode (serialize) the constructor in question.

learn by examples!
| constructor           | serialization                             |
|-----------------------|-------------------------------------------|
| `some#0x3f5476ca`     | 32-bit uint serialize from hex value      |
| `some$0101`           | serialize `0101` raw bits                 |
| `some`                | serialize `crc32(equation) \| 0x80000000` |
| `some#_` or `some$_`  | serialize nothing                         |
| `some#` or `some$`    | TL-B parsers will get it wrong            |


Tags may be given in either binary (after a dollar sign) or hexadecimal notation (after a hash sign). If a tag is not explicitly provided, TL-B parser must computes a default 32-bit constructor tag by hashing with crc32 algorithm the text of the “equation” with `| 0x80000000` defining this constructor in a certain fashion. Therefore, empty tags must be explicitly provided by `#_` or `$_`. 

All constructor names must be distinct, and constructor tags for the same type must constitute a prefix code (otherwise the deserialization would not be unique); i.e. no tag can be a prefix of any other.

This is an example from the [TonToken](https://github.com/akifoq/TonToken/blob/master/func/token/scheme.tlb) repository that shows  
us how to implement an internal message TL-B scheme:

```c++
extra#_ amount:Grams = Extra;

addr_std$10 anycast:(## 1) {anycast = 0}
      workchain_id:int8 address:bits256 = MsgAddrSmpl;

transfer#4034a3c0 query_id:uint64
    reciever:MsgAddrSmpl amount:Extra body:Any = Request;
```

In this example `transfer#4034a3c0` will be serialized as a 32 bit unsigned integer from hex value after hash sign(`#`). This meets the standard declaration of an `op` in the [Smart contract guidelines](https://ton.org/docs/#/howto/smart-contract-guidelines).

To meet the standard described in paragraph 5 of the [Smart contract guidelines](https://ton.org/docs/#/howto/smart-contract-guidelines), it is not enough for us to calculate the crc32. You can follow the following examples to define an `op` in requests or responses from smart contracts in a TL-B scheme:

```python
import binascii


def main():
    req_text = "some_request"
    req = format(binascii.crc32(bytes(req_text, "utf-8")) & 0x7fffffff, 'x')
    print(f"{req_text}#{req} = Request;")  # some_request#733d0d35 = Request;

    rsp_text = "some_response"
    rsp = format(binascii.crc32(bytes(rsp_text, "utf-8")) | 0x80000000, 'x')
    print(f"{rsp_text}#{rsp} = Response;")  # some_response#88b0eb8f = Response;


if __name__ == "__main__":
    main()
```


### Field definitions

The constructor and its optional tag are followed by field definitions. Each field definition is of the form `ident:type-expr`, where ident is an identifier with the name of the field16 (replaced by an underscore for anonymous fields), and type-expr is the field’s type. The type provided here is a type expression, which may include simple types or parametrized types with suitable parameters. Variables — i.e., the (identifiers of the) previously defined fields of types `#` (natural numbers) or `Type` (type of types) — may be used as parameters for the parametrized types. The serialization process recursively serializes each field according to its type, and the serialization of a value ultimately consists of the concatenation of bitstrings representing the constructor (i.e., the constructor tag) and the field values.

Some fields may be implicit. Their definitions are surrounded by curly
braces(`{`, `}`), which indicate that the field is not actually present in the serialization, but that its value must be deduced from other data (usually the parameters of the type being serialized). Example:

```c++
nothing$0 {X:Type} = Maybe X;
just$1 {X:Type} value:X = Maybe X;
```

Finally, some equalities/inequalities may be included in curly brackets as well. These are certain “equations”, which must be satisfied by the “variables” included in them. If one of the variables is prefixed by a tilde, its value will be uniquely determined by the values of all other variables participating in the equation (which must be known at this point) when the definition is processed from the left to the right. For example:

```c++
addr_std$10 anycast:(## 1) {anycast = 0}
      workchain_id:int8 address:bits256 = MsgAddrSmpl;
```

Some occurrences of “variables” (i.e., already-defined fields) are prefixed by a tilde(`~`). This indicates that the variable’s occurrence is used in the opposite way of the default behavior: in the left-hand side of the equation, it means that the variable will be deduced (computed) based on this occurrence, instead of substituting its previously computed value; in the right-hand side, conversely, it means that the variable will not be deduced from the type being serialized, but rather that it will be computed during the deserialization process. In other words, a tilde transforms an “input argument” into an “output argument”, and vice versa. 

For example, we can use this to write TL-B scheme for simple transaction in the TON with comment(which must be serialized as a sequence of cells):

```c++
empty#_ b:bits = Snake ~0;
cons#_ {n:#} b:bits next:^(Snake ~n) = Snake ~(n + 1);

op:#0 comment:Snake = Request;
```

A caret (`ˆ`) preceding a type `X` means that instead of serializing a value of type `X` as a bitstring inside the current cell, we place this value into a separate cell, and add a reference to it into the current cell. Therefore `ˆX` means “the type of references to cells containing values of type `X`”.

Parametrized type `#<= p` with `p : #` (this notation means `“p of type #”`, i.e., a natural number) denotes the subtype of the natural numbers type `#`, consisting of integers `0 ... p;` it is serialized into `[log2(p + 1)]` bits as an unsigned big-endian integer. Type `#` by itself is serialized as an unsigned 32-bit integer. Parametrized type `## b` with `b : #<=31` is equivalent to `#<= 2^b − 1`  (i.e., it is an unsigned b-bit integer). For example:

```c++
action_send_msg#0ec3c86d mode:(## 8) 
  out_msg:^(MessageRelaxed Any) = OutAction;
```

In this scheme `mode:(## 8)` will be serialized as 8-bit unsigned integer.

### Conditional (optional) fields

The serialization of the conditional fields is determined from the other, already specified fields.

For example(from [`block.tlb`](https://github.com/newton-blockchain/ton/blob/ae5c0720143e231c32c3d2034cfe4e533a16d969/crypto/block/block.tlb#L418)):

```c++
block_info#9bc7a987 version:uint32 
  not_master:(## 1) 
  after_merge:(## 1) before_split:(## 1) 
  after_split:(## 1) 
  want_split:Bool want_merge:Bool
  key_block:Bool vert_seqno_incr:(## 1)
  flags:(## 8) { flags <= 1 }
  seq_no:# vert_seq_no:# { vert_seq_no >= vert_seqno_incr } 
  { prev_seq_no:# } { ~prev_seq_no + 1 = seq_no } 
  shard:ShardIdent gen_utime:uint32
  start_lt:uint64 end_lt:uint64
  gen_validator_list_hash_short:uint32
  gen_catchain_seqno:uint32
  min_ref_mc_seqno:uint32
  prev_key_block_seqno:uint32
  gen_software:flags . 0?GlobalVersion
  master_ref:not_master?^BlkMasterInfo 
  prev_ref:^(BlkPrevInfo after_merge)
  prev_vert_ref:vert_seqno_incr?^(BlkPrevInfo 0)
  = BlockInfo;
```

In this example, the cell reference `^BlkMasterInfo` will be serialized only if `not_master` > 0. And the `GlobalVersion` will be serialized only if `flags == 0`.

### Namespaces

Available in the TL version from Telegram, but as it turned out, not used in TL-B

### Comments

Comments are the same as in C++
```
/* 
This is
a comment 
*/

// This is one line comment
```

### Library usage

You can use TL-B libraries to extend your documents and to avoid writing repetitive schemes. We have prepared a set of ready-made libraries that you can use. They are mostly based on block.tlb, but we have also added some combinators of our own.

- `tonstdlib.tlb`
- `tonextlib.tlb`
- `hashmap.tlb`

In TL-B libraries there is no concept of cyclic import. Just indicate the dependency on some other document (library) at the top of the document with the keyword `dependson`. For example:

file `mydoc.tlb`:
```c++
//
// dependson "libraries/tonstdlib.tlb"
//

op:uint32 data:Any = MsgBody;
something$0101 data:(Maybe ^MsgBody) = SomethingImportant;
```

In dependencies, you are required to specify the correct relative path. The example above is located in such a tree:

```
.
├── mydoc.tlb
├── libraries
│   ├── ...
│   └── tonstdlib.tlb
└── ...
```

### IDE Support

The [intellij-ton](https://github.com/andreypfau/intellij-ton) plugin supports Fift, FunC and also TL-B.  
The TL-B grammar is described in the [TlbParser.bnf](https://github.com/andreypfau/intellij-ton/blob/main/src/main/grammars/TlbParser.bnf) file.

### Useful sources

- [Telegram Open Network Virtual Machine](https://ton.org/docs/tvm.pdf)
- [A description of an older version of TL](https://core.telegram.org/mtproto/TL)
- [block.tlb](https://github.com/newton-blockchain/ton/blob/master/crypto/block/block.tlb)
