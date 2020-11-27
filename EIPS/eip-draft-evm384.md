---
eip: <to be assigned>
title: EVM384 opcodes
author: Alex Beregszaszi (@axic), Paul Dworzanski (@poemm), Jared Wasinger (@jwasinger), Casey Detrio (@cdetrio)
discussions-to: https://ethereum-magicians.org/t/evm384-feedback-and-discussion/4533
status: Draft
type: Standards Track
category: Core
created: 2020-10-01
---

## Abstract

Introducing three opcodes to speed up 384-bit computations. They can be useful for efficiently implementing elliptic curve operations
on up to 384-bit curves, this includes BLS12-381 and BLS12-377, among others.

## Motivation

The EVM is perceived to be too slow for computationally heavy tasks and this has resulted in introducing several precompiles over the years.
[Prior work](https://notes.ethereum.org/@axic/evm384#History-of-precompiles) has found that this assumption has been challenged a few times.
This motivated the work to see where the BLS12-381 curve operations

 - User-deployed fast cryptography for all algorithms with bottlenecks in modular arithmetic over a <= 384-bit modulus. For example, operations on BLS12-381 and BLS12-377.
 - Modular exponentiation is common in cryptography, for example in RSA-based cryptosystems. The MODEXP precompile attempts to enable fast on-chain modular exponentiation. Unfortunately, it is difficult to meter modular exponentiation, since it must use a generic algorithm to handle arbitrary inputs, and the gas cost must consider worst-case runtime. Furthermore, for modular exponentiation, the generic algorithm is rarely the [most optimal](https://en.wikipedia.org/wiki/Modular_exponentiation#Minimum_multiplications). Fortunately, the bottleneck of modular exponentiation is modular multiplication, which is covered by EVM384 opcode `mulmodmont384`. With this bottleneck covered, the user can deploy arbitrary modular exponentiation algorithms which should be much more efficient than the MODEXP precompile. But for the desired RSA 3072-bit calculations, we would need mulmodmont3072, which would have to be proposed separately from EVM384.
 - polynomial evaluation: STARKS require fast on-chain polynomial evaluation. Current STARKs use a 256-bit modulus, for which EVM opcodes MULMOD, ADDMOD, and SUBMOD can be used. But the opcode MULMOD is generic and can be expensive in the worst case. MULMOD is currently underpriced, and we expect EVM384 opcodes to be less expensive, even with a 256-bit modulus.


The motivation is critical for EIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.

https://notes.ethereum.org/@axic/evm384-preview
https://notes.ethereum.org/@axic/evm384
https://notes.ethereum.org/@poemm/evm384-interface-update

## Specification

We use notation from the yellowpaper.

| value    | Mnemonic      | delta   | alpha   | Description|
| -------- | --------      | ---     | ---     | ---        |
| 0xc0     | ADDMOD384     | (below) | (below) | (below)    |
| 0xc1     | SUBMOD384     | (below) | (below) | (below)    |
| 0xc2     | MULMODMONT384 | (below) | (below) | (below)    |

Recall from the yellowpaper that delta is number of elements popped from stack and alpha is number of elements pushed to stack. In the descriptions below, we replace yellowpaper symbols $𝛍_s$, $𝛍_m$, and $𝛍_i$ with `stack`, `memory`, and `memoryLength`, respectively. In particular, `stack[0]` refers to the first item popped from the stack, `stack[1]` is the second item popped from the stack, and so on. `stack'[0]` and `stack'[1]` are the top and second-to-top item pushed to the stack by the operation. `memory[i..i+48]` refers to the memory bytes from byte `i` to `i+48`. `memoryLength` is the current number of 32-byte chunks in memory. Finally, function `M(current_memoryLength, start_byte_offset, bytelength_accessed)` is the "memory expansion for range function" from the yellowpaper H.1. 

#### ADDMOD384 | alpha = 1 | delta = 0 | Description:
```
out_offset = stack[0][16:20]
x_offset = stack[0][20:24]
y_offset = stack[0][24:28]
mod_offset = stack[0][28:32]  i.e. least-significant bytes of the big-endian stack item.

Where each offset is a big-endian unsigned 32-bit integer.

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
out_offset = stack[0][16:20]
x_offset = stack[0][20:24]
y_offset = stack[0][24:28]
mod_offset = stack[0][28:32]

Where each offset is a big-endian unsigned 32-bit integer.

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
out_offset = stack[0][16:20]
x_offset = stack[0][20:24]
y_offset = stack[0][24:28]
mod_and_inv_offset = stack[0][28:32]

Where each offset is a big-endian unsigned 32-bit integer.

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

These instructions were identified as 

### Packed arguments

...

### Endianess

...

## Backwards Compatibility

This EIP introduces new opcodes which did not exists previously. Already deployed contracts using these opcodes could change their behaviour after this EIP.

## Test Cases

TBA

## Implementation

TBA

## Security Considerations

TBA

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
