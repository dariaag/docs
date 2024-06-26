---
sidebar_position: 80
description: >-
  Typegen tool
---

# Typegen

A _typegen_ is a tool for generating utility code for technology-specific operations such as decoding.

Solana typegen:

- decodes program data from IDL
- handles direct calls to program instructions
- exposes useful constants such as event logs and instructions

Install with

```bash
  $ npm install @subsquid/solana-typegen
```

The `squid-solana-typegen` tool generates TypeScript facades for EVM transactions, logs and instruction queries.

The generated facades are assumed to be used by squids indexing Solana data.

The tool takes a JSON IDLs as an input. Those can be specified in three ways:

1. as a plain JSON file(s):

   ```bash
   $ npx squid-solana-typegen src/abi whirlpool.json
   ```

   If you use this option, you can also place your JSON IDLs to the `abi` folder and run

   ```bash
   $ sqd typegen
   ```

## Usage

The generated utility modules have three intended uses:

1. Constants: Solana instruction discriminators:

   ```ts
   // generated by evm-typegen
   import * as whirlpool from "./abi/whirlpool";

   const dataSource = new DataSourceBuilder()
     .setRpc(
       process.env.SOLANA_NODE == null
         ? undefined
         : {
             client: new SolanaRpcClient({
               url: process.env.SOLANA_NODE,
               // rateLimit: 100 // requests per sec
             }),
             strideConcurrency: 10,
           }
     )
     .addInstruction({
       where: {
         programId: [whirlpool.programId], // where executed by Whirlpool program
         d8: [whirlpool.swap.d8], // have first 8 bytes of .data equal to swap descriptor
         isCommitted: true, // where successfully committed
       },
     })
     .build();
   ```

2. Decoding of Solana inner instructions

```ts
import * as tokenProgram from "./abi/tokenProgramIDL";
for (let block of blocks) {
  for (let ins of block.instructions) {
    // https://read.cryptodatabytes.com/p/starter-guide-to-solana-data-analysis
    if (ins.programId === whirlpool.programId && ins.d8 === whirlpool.swap.d8) {
      let exchange = new Exchange({
        id: ins.id,
        slot: block.header.slot,
        tx: ins.getTransaction().signatures[0],
        timestamp: new Date(block.header.timestamp * 1000),
      });

      assert(ins.inner.length == 2);
      let srcTransfer = tokenProgram.transfer.decode(ins.inner[0]); //decoding inner instruction
      let destTransfer = tokenProgram.transfer.decode(ins.inner[1]); //decoding inner instruction
    }
  }
}
```
