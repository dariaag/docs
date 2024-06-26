---
sidebar_position: 10
title: General settings
description: >-
  Data sourcing and metrics
---

# General settings

:::tip
The method documentation is also available inline and can be accessed via suggestions in most IDEs.
:::

The following setters configure the global settings of `DataSourceBuilder` for Solana Procesor. They return the modified instance and can be chained.

Certain configuration methods are required:

- one or both of [`setGateway()`](#set-gateway) and [`setRpcEndpoint()`](#set-rpc-endpoint)
- [`setFinalityConfirmation()`](#set-finality-confirmation) whenever [RPC ingestion](/sdk/resources/basics/unfinalized-blocks) is enabled, namely when
  - a RPC endpoint was configured with [`setRpcEndpoint()`](#set-rpc-endpoint)
  - RPC ingestion has **NOT** been explicitly disabled by calling [`setRpcDataIngestionSettings({ disabled: true })`](#set-rpc-data-ingestion-settings)

Here's how to choose the data sources depending on your use case:

- If you need real-time data use both [`setGateway()`](#set-gateway) and [`setRpcEndpoint()`](#set-rpc-endpoint). The processor will obtain as much data as is currently available from the dataset, then switch to ingesting recent data from the RPC endpoint.
- If you can tolerate your data being several thousands of blocks behind the chain head, you do not want to use a RPC endpoint, use [`setGateway()`](#set-gateway) only.

- If you can tolerate a latency of a few thousands of blocks, you can disable RPC ingestion with [`setRpcDataIngestionSettings({ disabled: true })`](#set-rpc-data-ingestion-settings). In this scenario RPC will only be used for the queries you explicitly make in your code.

### `setGateway(url: string | GatewaySettings)` {#set-gateway}

Adds a [Subsquid Network](/subsquid-network) data source. The argument is either a string URL of a dataset served by a Subsquid Network gateway or

```ts
{
  url: string // dataset URL
  requestTimeout?: number // in milliseconds
}
```

### `setRpcEndpoint(rpc: ChainRpc)` {#set-rpc-endpoint}

Adds a RPC data source. If added, it will be used for

- [RPC ingestion](/sdk/resources/basics/unfinalized-blocks) (unless explicitly disabled with [`setRpcDataIngestionSettings()`](#set-rpc-data-ingestion-settings))

A node RPC endpoint can be specified as a string URL or as an object:

```ts
type ChainRpc =
  | string
  | {
      url: string; // http, https, ws and wss are supported
      capacity?: number; // num of concurrent connections, default 10
      maxBatchCallSize?: number; // default 100
      rateLimit?: number; // requests per second, default is no limit
      requestTimeout?: number; // in milliseconds, default 30_000
    };
```

Setting `maxBatchCallSize` to `1` disables batching completely.

:::tip
We recommend using private endpoints for better performance and stability of your squids. For Subsquid Cloud deployments you can use the [RPC proxy](/cloud/reference/rpc-proxy). If you use an external private RPC, keep the endpoint URL in a [Cloud secret](/cloud/resources/env-variables#secrets).
:::

### `setDataSource(ds: {archive?: string, chain?: ChainRpc})` (deprecated) {#set-data-source}

Replaced by [`setGateway()`](#set-gateway) and [`setRpcEndpoint()`](#set-rpc-endpoint).

### `setRpcDataIngestionSetting(settings: RpcDataIngestionSettings)` {#set-rpc-data-ingestion-settings}

Specify the [RPC ingestion](/sdk/resources/basics/unfinalized-blocks) settings.

```ts
type RpcDataIngestionSettings = {
  disabled?: boolean;
  preferTraceApi?: boolean;
  useDebugApiForStateDiffs?: boolean;
  debugTraceTimeout?: string;
  headPollInterval?: number;
  newHeadTimeout?: number;
};
```

Here,

- `disabled`: Explicitly disables data ingestion from an RPC endpoint.
- `preferTraceApi`: By default, [`debug_traceBlockByHash`](https://geth.ethereum.org/docs/interacting-with-geth/rpc/ns-debug#debugtraceblockbyhash) is used to obtain [call traces](../traces). This flag instructs the processor to utilize [`trace_` methods](https://openethereum.github.io/JSONRPC-trace-module) instead. This setting is only effective for finalized blocks.
- `headPollInterval`: Poll interval for new blocks in milliseconds. Poll mechanism is used to get new blocks via HTTP connections. Default: 5000.
- `newHeadTimeout`: When ingesting from a websocket, this setting specifies the timeout in milliseconds after which the connection will be reset and subscription re-initiated if no new blocks were received. Default: no timeout.

### `setBlockRange({from: number, to?: number})` {#set-block-range}

Limits the range of blocks to be processed. When the upper bound is specified, processor will terminate with exit code 0 once it reaches it.

Note that block ranges can also be specified separately for each data request. This method sets global bounds for all block ranges in the configuration.

### `includeAllBlocks(range?: {from: number, to?: number})` {#include-all-blocks}

By default, processor will fetch only blocks which contain requested items. This method modifies such behavior to fetch all chain blocks. Optionally a range of blocks can be specified for which the setting should be effective.
