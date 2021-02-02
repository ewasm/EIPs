---
eip: <to be assigned>
title: EVM384 (v7) instructions
author: Alex Beregszaszi (@axic), Paul Dworzanski (@poemm), Jared Wasinger (@jwasinger), Casey Detrio (@cdetrio), Paweł Bylica (@chfast)
discussions-to: https://ethereum-magicians.org/t/evm384-feedback-and-discussion/4533
status: Draft
type: Standards Track
category: Core
created: 2020-10-01
---

## Abstract

Introducing three opcodes to speed up <=384-bit computations. They can be useful for efficiently implementing elliptic curve operations
on up to 384-bit curves, this includes BLS12-381 and BLS12-377, among others.

## Motivation

The EVM is perceived to be too slow for computationally heavy tasks and this has resulted in introducing several precompiles over the years.
[Prior work](https://notes.ethereum.org/@axic/evm384#History-of-precompiles) has found that this assumption has been challenged a few times.

The EVM384 project was motivated by bringing support for BLS12-381 without a precompile. The prior articles include an [introduction](https://notes.ethereum.org/@axic/evm384)
and several updates ([2](https://notes.ethereum.org/@poemm/evm384-interface-update), [3](https://notes.ethereum.org/@poemm/evm384-update3),
[4](https://notes.ethereum.org/@poemm/evm384-update4), [5](https://notes.ethereum.org/@poemm/evm384-update5)) explaining the background, various considerations,
and results. We suggest referring to them for historical details.

## Specification

We use notation from the Yellow Paper.

| value    | Mnemonic      | delta   | alpha   | Description|
| -------- | --------      | ---     | ---     | ---        |
| 0xc0     | ADDMOD384     | (below) | (below) | (below)    |
| 0xc1     | SUBMOD384     | (below) | (below) | (below)    |
| 0xc2     | MULMODMONT384 | (below) | (below) | (below)    |

Recall from the Yellow Paper that delta is number of elements popped from stack and alpha is number of elements pushed to stack.
In the descriptions below, we replace Yellow Paper symbols $𝛍_s$, $𝛍_m$, and $𝛍_i$ with `stack`, `memory`, and `memoryLength`, respectively.
In particular, `stack[0]` refers to the first item popped from the stack, `stack[1]` is the second item popped from the stack, and so on.
`stack'[0]` and `stack'[1]` are the top and second-to-top item pushed to the stack by the operation. `memory[i..i+48]` refers to the memory
bytes from byte `i` to `i+48`. `memoryLength` is the current number of 32-byte chunks in memory.
Finally, function `M(current_memoryLength, start_byte_offset, bytelength_accessed)` is the "memory expansion for range function" from the Yellow Paper H.1. 

#### ADDMOD384 | alpha = 1 | delta = 0 | Description:
```
out_offset = stack[0][24:26]
x_offset = stack[0][26:28]
y_offset = stack[0][28:30]
mod_offset = stack[0][30:32]  i.e. least-significant bytes of the big-endian stack item.

Where each offset is a big-endian unsigned 16-bit integer.

x = memory[x_offset:x_offset+48]
y = memory[y_offset:y_offset+48]
mod = memory[mod_offset:mod_offset+48]

memory[out_offset:out_offset+48] = [ x + y        if x+y < mod
                                   [ x + y - mod  otherwise

Where x, y, mod, and result are integers of at-most 384-bits stored in memory as little-endian.

Must also check memory length and possibly grow the memory.
memoryLength = M( M( M( M( memoryLength, x_offset, 48 ), y_offset, 48 ), out_offset, 48 ), mod_offset, 48 )
```

#### SUBMOD384 | alpha = 1 | delta = 0 | Description:
```
out_offset = stack[0][24:26]
x_offset = stack[0][26:28]
y_offset = stack[0][28:30]
mod_offset = stack[0][30:32]

Where each offset is a big-endian unsigned 16-bit integer.

x = memory[x_offset:x_offset+48]
y = memory[y_offset:y_offset+48]
mod = memory[mod_offset:mod_offset+48]

memory[out_offset:out_offset+48] = [ x - y        if x-y >= 0
                                   [ x - y + mod  otherwise

Where x, y, mod, and result are integers of at-most 384-bits stored in memory as little-endian.

Must also check memory length and possibly grow the memory.
memoryLength = M( M( M( M( memoryLength, x_offset, 48 ), y_offset, 48 ), out_offset, 48 ), mod_offset, 48 )
```

#### MULMODMONT384 | alpha = 1 | delta = 0 | Description:
```
out_offset = stack[0][24:26]
x_offset = stack[0][26:28]
y_offset = stack[0][28:30]
mod_and_inv_offset = stack[0][30:32]

Where each offset is a big-endian unsigned 16-bit integer.

x = memory[x_offset:x_offset+48]
y = memory[y_offset:y_offset+48]
mod = memory[mod_rinv_offset:mod_rinv_offset+48]
r_inv = memory[mod_rinv_offset+48:mod_rinv_offset+56]

memory[out_offset:out_offset+48] =  (x * y * r_inv) % mod
Where the multiplication operation is montgomery multiplication, even if modulus is even.

Where x, y, mod, and result are integers of at-most 384-bits stored in memory as little-endian. Similarly, mod_r is an integer of at-most 64-bits stored in memory as little-endian.

Must also check memory length and possibly grow the memory.
memoryLength = M( M( M( M( memoryLength, x_offset, 48 ), y_offset, 48 ), out_offset, 48 ), mod_and_inv_offset, 56 )
```

## Rationale

### Instructions

Our design was initially motivated by enabling the implementation of BLS12-381 operations, which only requires addition, subtraction, and exponentiation.
We have [identified some candidates](https://notes.ethereum.org/@poemm/evm384-update5#INVERSEMOD384-and-SQUAREROOTMOD384) for further extension, but they
are not a requirement for BLS12-381. The other uses cases we have looked at were implementable using these instructions too. 

Should the need arise, it would be possible to introduce other operators using the design of this EIP.

### Packed arguments

In [earlier designs](https://notes.ethereum.org/@poemm/evm384-interface-update) we have explored instructions taking each of the parameters as individual stack items.
While this matches the design of other EVM opcodes, the stack manipulation overhead is not justifiable. 

Packing the parameters into a single stack slot not only removes this overhead, but introduces a beneficial restriction by limiting the addressable memory.

### Endianess

The EVM treats data in a big endian encoding. We have [noted](https://notes.ethereum.org/@poemm/evm384-interface-update#Conclusion-and-Next-Steps) that EVM384 decided
to treat data (the values in memory) in little endian encoding. The majority of today's CPUs on which Ethereum clients run are natively little endian. The implementation
of our instructions take advantage of this fact. Benchmarks have shown that measurable slowdown occurs implementing these instructions to deal with big endian input.

Whiel we have not removed this from consideration, the interface as it stands operates on little endian data.

### Gas costs

This EIP currently uses the Model A of [proposed costs](https://notes.ethereum.org/@poemm/evm384-update5#Proposed-EVM384-Gas-Costs). The rationale 
for these specific values is explained in [the appendix](https://notes.ethereum.org/@poemm/evm384-update5#Appendix-A-Model-A-%E2%80%94-The-BaseOperation-Gas-Model),
including a [security analysis](https://notes.ethereum.org/@poemm/EVM384SecurityAnalysis).

## Backwards Compatibility

This EIP introduces new instructions which did not exists previously. Already deployed contracts using these opcodes could change their behaviour after this EIP.

## Test Cases

TBA

## Implementation

TBA

## Security Considerations

TBA

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
