---
sidebar_position: 60
description: >-
  Fine-tuning data requests with setFields()
---

# Field selection

#### `setFields(options)` {#set-fields}

Set the fields to be retrieved for data items of each supported type. The `options` object has the following structure:

```ts
{
  signatures?:         // field selector for logs
  err?: // field selector for transactions
}
```

Every field selector is a collection of boolean fields, typically mapping one-to-one to the fields of data items within the batch context [iterables](/sdk/reference/processors/solana-batch/context-interfaces). Defining a field of a field selector of a given type and setting it to true will cause the processor to populate the corresponding field of all data items of that type. Here is a definition of a processor that requests `signatures` and `err` fields for transactions:

```ts
const dataSource = new DataSourceBuilder().setFields({
  block: {
    // block header fields
    timestamp: true,
  },
  transaction: {
    // transaction fields
    signatures: true,
  },
});
```

Same fields will be available for all data items of any given type, including nested items. Suppose we used the processor defined above to subscribe to some transactions as well as some logs, and for each log we requested a parent transaction:

```ts
processor
  .addLog({
    // some log data requests
    transaction: true,
  })
  .addTransaction({
    // some transaction data requests
    logs: true,
  });
```

As a result, `signatures` and `err` fields would be available both within the transaction items of the `transactions` iterable of [block data](/sdk/reference/processors/solana-batch/context-interfaces) and within the transaction items that provide parent transaction information for the logs:

```ts
run(dataSource, database, async (ctx) => {
  let blocks = ctx.blocks.map(augmentBlock);
  for (let block of blocks) {
    for (let txn of block.transactions) {
      let sig = txn.signature; // OK
    }
    for (let log of block.logs) {
      let parentTxSig = log.transaction.signature; // also OK!
    }
  }
});
```

Some data fields, like `signatures` for transactions, are enabled by default but can be disabled by setting a field of a field selector to `false`. For example, this code will not compile:

```ts
const dataSource = new DataSourceBuilder().setFields({
  transaction: {
    // transaction fields
    signatures: false,
  },
});

run(dataSource, database, async (ctx) => {
  for (let block of ctx.blocks) {
    for (let txn of block.transactions) {
      let signatures = txn.signatures; // ERROR: no such field
    }
  }
});
```

Disabling unused fields will improve sync performance, as the disabled fields will not be fetched from the Subsquid Network gateway.

## Data item types and field selectors

:::tip
Most IDEs support smart suggestions to show the possible field selectors. For VS Code, press `Ctrl+Space`.
:::

Here we describe the data item types as functions of the field selectors. Unless otherwise mentioned, each data item type field maps to the eponymous field of its corresponding field selector. Item fields are divided into three categories:

- Fields that can be disabled by `setFields()`. E.g. a `signatures` field will be fetched for transactions by default, but can be disabled by setting `signatures: false` within the `log` field selector.

[//]: # "!!!! update with final defaults and capabilities"

### Logs

`Log` data items may have the following fields:

```ts
Log {
  // independent of field selectors
  programId: string
  kind: string
  message: string

}
```

See the [block headers section](#block-headers) for the definition of `BlockHeader` and the [transactions section](#transactions) for the definition of `Transaction`.

### Transactions

`Transaction` data items may have the following fields:

```ts
Transaction {
  // independent of field selectors
  signatures: Base58Bytes[]
  err: null | object

  // can be requested with field selectors
    transactionIndex: number
    version: 'legacy' | number
    // transaction message
    accountKeys: Base58Bytes[]
    addressTableLookups: AddressTableLookup[]
    numReadonlySignedAccounts: number
    numReadonlyUnsignedAccounts: number
    numRequiredSignatures: number
    recentBlockhash: Base58Bytes
    signatures: Base58Bytes[]
    // meta fields
    err: null | object
    computeUnitsConsumed: bigint
    fee: bigint
    loadedAddresses: {
        readonly: Base58Bytes[]
        writable: Base58Bytes[]
    }
    hasDroppedLogMessages: boolean
}
```

<!-- `status` field contains the value returned by [`eth_getTransactionReceipt`](https://geth.ethereum.org/docs/interacting-with-geth/rpc/batch): `1` for successful transactions, `0` for failed ones and `undefined` for chains and block ranges not compliant with the post-Byzantinum hard fork EVM specification (e.g. 0-4,369,999 on Ethereum).

`type` field is populated similarly. For example, on Ethereum `0` is returned for Legacy txs, `1` for EIP-2930 and `2` for EIP-1559. Other networks may have a different set of types.

See the [block headers section](#block-headers) for the definition of `BlockHeader`. -->

### Instruction

`Instruction` data items may have the following fields:

```ts
Instruction {
  // independent of field selectors
    programId: Base58Bytes
    accounts: Base58Bytes[]
    data: Base58Bytes
    /**
     * `true` when transaction completed successfully, `false` otherwise
     */
    isCommitted: boolean

  // can be enabled with field selectors
    transactionIndex: number
    instructionAddress: number[]
    // execution result extracted from logs
    computeUnitsConsumed?: bigint
    error?: string
    hasDroppedLogMessages: boolean
}
```

<!-- The meaning of the `kind` field values is as follows:

- `'='`: no change has occured;
- `'+'`: a value was added;
- `'*'`: a value was changed;
- `'-'`: a value was removed.

The values of the `key` field are regular hexadecimal contract storage key strings or one of the special keys `'balance' | 'code' | 'nonce'` denoting ETH balance, contract code and nonce value associated with the state diff.

See the [block headers section](#block-headers) for the definition of `BlockHeader` and the [transactions section](#transactions) for the definition of `Transaction`. -->

### LogMessage

`LogMessage` data items may have the following fields:

```ts
 LogMessage {
    // independent of field selectors
    programId: Base58Bytes
    kind: 'log' | 'data' | 'other'
    message: string
    // can be enabled with field selectors
    transactionIndex: number
    logIndex: number
    instructionAddress: number[]
}
```

### TokenBalances

Field selection for token balances data items is more nuanced because depending on the subtype of the token balance some fields may be `undefined`. `PostTokenBalance` and `PreTokenBalance` both represent token balances, however `PreTokenBalance` will have `postProgramId, postMint, postDecimals, postOwner and postAmount` as `undefined`.

`PostTokenBalance` will have `preProgramId, preMint, preDecimals, preOwner and preAmount` as `undefined`.

`TokenBalance` data items may have the following fields:

```ts
 TokenBalance {
    // independent of field selectors
    preProgramId?: Base58Bytes
    preMint: Base58Bytes
    preDecimals: number
    preOwner?: Base58Bytes
    preAmount: bigint

    postProgramId?: Base58Bytes
    postMint: Base58Bytes
    postDecimals: number
    postOwner?: Base58Bytes
    postAmount: bigint
    // can be enabled by field selectors
    transactionIndex: number
    account: Base58Bytes
}
```

### Block headers

`BlockHeader` data items may have the following fields:

```ts
BlockHeader{
  // independent of field selectors
  slot: number
  parentSlot: number
  timestamp: number
  // can be enabled by field selectors
  hash: Base58Bytes
  height: number
  parentHash: Base58Bytes
}
```

## A complete example

```ts
import { run } from "@subsquid/batch-processor";
import { augmentBlock } from "@subsquid/solana-objects";
import { DataSourceBuilder, SolanaRpcClient } from "@subsquid/solana-stream";
import { TypeormDatabase } from "@subsquid/typeorm-store";
import * as whirlpool from "./abi/whirpool";

const gravatarRegistryContract = "0x2e645469f354bb4f5c8a05b3b30a929361cf77ec";
const gravatarTokenContract = "0xac5c7493036de60e63eb81c5e9a440b42f47ebf5";

const dataSource = new DataSourceBuilder()
  .setGateway("https://v2.archive.subsquid.io/network/solana-mainnet")
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
  .setBlockRange({ from: 240_000_000 });
setFields({
  block: {
    // block header fields
    timestamp: true,
  },
  transaction: {
    // transaction fields
    signatures: true,
  },
  instruction: {
    // instruction fields
    programId: true,
    accounts: true,
    data: true,
  },
  tokenBalance: {
    // token balance record fields
    preAmount: true,
    postAmount: true,
    preOwner: true,
    postOwner: true,
  },
})
  .addInstruction({
    // select instructions, that:
    where: {
      programId: [whirlpool.programId], // where executed by Whirlpool program
      d8: [whirlpool.swap.d8], // have first 8 bytes of .data equal to swap descriptor
      isCommitted: true, // where successfully committed
    },
    include: {
      innerInstructions: true, // inner instructions
      transaction: true, // transaction, that executed the given instruction
      transactionTokenBalances: true, // all token balance records of executed transaction
    },
  })
  .build();

processor.run(new TypeormDatabase(), async (ctx) => {
  // Simply output all the items in the batch.
  // It is guaranteed to have all the data matching the data requests,
  // but not guaranteed to not have any other data.
  ctx.log.info(ctx.blocks, "Got blocks");
});
```
