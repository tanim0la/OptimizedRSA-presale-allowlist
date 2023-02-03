# OptimizedRSA-presale-allowlist

This is a [RareSkills.io](https://RareSkills.io) project from the [Solidity Bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps-solidity) to allowlist addresses far more efficiently than ECDSA or Merkle Trees.

For a detailed breakdown see Jeffrey Scholz's medium post [here](https://medium.com/donkeverse/hardcore-gas-savings-in-nft-minting-part-2-signatures-vs-merkle-trees-917c43c59b07)

<hr>

I was able to shave off `191` gas from each test when the function `verifySignature` is called. Here are the changes I made:

#

I noticted the `add` opcode was adding two static values, so I removed the `add` opcode and replaced with the known results; `0xa0` and `0xc0` which saved 6 gas.

Before (line 122 - 123)

```
    mstore(add(0x80, 0x20), 0x20)
    mstore(add(0x80, 0x40), sig.length)
```

After (line 122 - 123)

```
    mstore(0xa0, 0x20)
    mstore(0xc0, sig.length)
```

#

The variables `modPos` and `callDataSize` were replaced with `add(sig.length, 0x100)` and `add(0x80, mul(sig.length, 2))` respectively in `extcodecopy` and `staticcall` opcodes because the variables were use just once.

Before (line 133 - 161)

```
    let modPos := add(0xe0, add(sig.length, 0x20))
    let callDataSize := add(0x80, mul(sig.length, 2))
    extcodecopy(_metamorphicContractAddress, modPos, 0x33, sig.length)
    staticcall(gas(), 0x05, 0x80, callDataSize, 0x80, sig.length)
```

After (line 136 - 157)

```
    extcodecopy(_metamorphicContractAddress, add(sig.length, 0x100), 0x33, sig.length)
    staticcall(gas(), 0x05, 0x80, add(0x80, mul(sig.length, 2)), 0x80, sig.length)
```

#

The opcode `mul(sig.length, 2)` was replaced with `add(sig.length, sig.length)` because `mul` cost 5 gas and `add` cost 3 gas.

Before (line 157)

```
    staticcall(gas(), 0x05, 0x80, add(0x80, mul(sig.length, 2)), 0x80, sig.length)
```

After (line 157)

```
    staticcall(gas(), 0x05, 0x80, add(0x80, add(sig.length, sig.length)), 0x80, sig.length)
```

#

I noticed the for loop started its check at position `0x80`, so I was able to skip the first two loops using the `or` opcode to compare each byte at positon `0x80` and `0xa0`, if the result is 0, it means the condition is false and the statement in the block does not execute.

Before (line 172 - 178)

```
    for { let i := 1 } lt(i, chunksToCheck) { i := add(i, 1) }
     {
          if  mload(add(0x60, mul(i, 0x20)))
          {
              revert(0, 0)
          }
     }
```

After (line 168 - 178)

```
    if or(mload(0x80), mload(0xa0))
     {
          revert(0, 0)
     }

    for { let i := 3 } lt(i, chunksToCheck) { i := add(i, 1) }
     {
          if  mload(add(0x60, mul(i, 0x20)))
          {
              revert(0, 0)
          }
     }
```

#

The variable `decodedSig` was replaced with `mload(add(0x60, sig.length))` and I used `eq` opcode to check if caller is equal to the decoded signature, then I store the value returned from `eq` opcode to the postion of returndatasize since it cost only 2 gas.

Before (line 184 - 192)

```
    let decodedSig := mload(add(0x60, sig.length))
    if eq(caller(), decodedSig)
    {
         // Return true
         mstore(0x00, 0x01)
         return(0x00, 0x20)
    }
    // Else Return false
    mstore(0x00, 0x00)
    return(0x00, 0x20)
```

After (line 185 - 186)

```
    mstore(returndatasize(), eq(caller(), mload(add(0x60, sig.length))))
    return(returndatasize(), 0x20)
```

#

### Approach before optimizing:

- RSA 896 bit Metamorphic (Gas: 27,040)
- RSA 960 bit Metamorphic (Gas: 27,115)
- RSA 1024 bit Metamorphic (Gas: 27,311)
- RSA 2048 bit Metamorphic (Gas: 29,901)

### Approach after optimizing:

- RSA 896 bit Metamorphic (Gas: 26,849)
- RSA 960 bit Metamorphic (Gas: 26,924)
- RSA 1024 bit Metamorphic (Gas: 27,120)
- RSA 2048 bit Metamorphic (Gas: 29,710)

RSA 2048 bit gas can be beaten down to `29,534` gas if the counter `i` in the for loop is increment by 2, but doing this will skip some chunks to check which breaks the cryptography.

<hr>

## Tests

Run tests: `npx hardhat test`

<img width="800" alt="Screenshot 2023-01-31 at 22 09 54" src="https://user-images.githubusercontent.com/36541366/215883887-92da810c-8f88-47bb-ac5c-6fa9d89c70f1.png">

## Working with the repo:

Clone repo: `git clone <https/ssh string>`

Install packages: `npm i`
