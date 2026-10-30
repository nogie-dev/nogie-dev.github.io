+++
title = "MetaMask Gas Station: transaction submission paths and IP traceability"
date = 2026-10-30
description = "A source-code analysis of MetaMask Gas Station submission paths, the information sent to Sentinel, and the conditions needed to link a transaction to a connection IP."
draft = false

[taxonomies]
tags = ["RealWorld"]
+++

## The incident that prompted this analysis

While analyzing the Binance compromise, Justin Kalland reported signs of MetaMask Gas Station use by a wallet identified as belonging to the attacker. According to [his post](https://x.com/justinkalland/status/2115946980665151955), Gas Station sent ETH to the wallet, which then approved the Kyber router to spend USDT and paid USDT to Gas Station, all within the same block. The wallet reportedly paid for gas with tokens despite already holding ETH.

Kalland also noted that the ETH top-up matched the combined maximum gas budgets of the next two transactions, down to the wei. He presented the transaction order within the block and this exact amount match as evidence of Gas Station use.

The official MetaMask account [replied](https://x.com/MetaMask/status/2115977377989955778): “Our security team is aware of this report and are investigating”.

<figure>
<img src="/img/metamask-gas-station/metamask-official-reply.jpg" alt="MetaMask's official reply to the report" width="332" height="153" style="height: auto;" loading="lazy">
<figcaption>MetaMask's official reply to the report</figcaption>
</figure>

This analysis follows the transaction submission path. If the wallet sent requests to MetaMask's servers while using Gas Station, those servers' connection records would also be relevant to an investigation. MetaMask's source code shows where these requests are sent and what information they contain.

## From MetaMask Gas Station to block inclusion

Sentinel is a backend API that MetaMask Wallet uses to request transaction simulations, submit transactions, and check their processing status. For the Gas Station paths examined here, a transaction that is successfully processed and included in a block follows this broad flow:

MetaMask Wallet prepares and signs the transaction or delegation → sends a request to the Sentinel API → the backend handles submission → the transaction is included in an Ethereum block.

The MetaMask Sentinel API server address used for Ethereum Mainnet is `https://tx-sentinel-ethereum-mainnet.api.cx.metamask.io/`.

The [JSON-RPC client code](https://github.com/MetaMask/core/blob/b8d07a5a7731c565ad6c49b555b21b7cceedce61/packages/sentinel-api-service/src/sentinel-api-service.ts#L392-L408) used by 7702 Relay constructs requests as follows:

```typescript
const response = await this.#fetch(url, {
  method: 'POST',
  headers: {
    ...headers,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    id: '1',
    jsonrpc: '2.0',
    method,
    params,
  }),
});
```

This code constructs the HTTP request sent to Sentinel. The `method` field names the API method, and `params` contains the request data. For 7702 Relay submission, the method is `eth_sendRelayTransaction`.

The submission request goes to Sentinel. At the HTTPS entry point where TLS terminates, the receiver can observe the connection peer's IP address and read the request contents. If the request data identifies the user's wallet, the receiver could associate the wallet's request with that connection IP.

<figure>
<img src="/img/metamask-gas-station/ip-trace-sketch-en-revised.png" alt="Standard RPC and MetaMask Gas Station: submission paths and IP observation points" width="1422" height="1106" style="height: auto;" loading="lazy">
<figcaption>Standard RPC and MetaMask Gas Station: submission paths and IP observation points</figcaption>
</figure>

The solid arrows in this comparison represent request paths verified in the client code. The dashed arrows represent server-side processing that was not directly verified. The yellow boxes illustrate possible correlations if the required records are retained and can be linked. Block inclusion remains conditional on the transaction being accepted and selected.

## Standard RPC submission and Gas Station submission

In a standard submission, the wallet sends `eth_sendRawTransaction` to its configured RPC endpoint. The two Gas Station paths examined here send their submission requests to Sentinel. MetaMask chooses a submission path according to conditions such as the account type and network support.

The orange boxes mark points where a server can observe both the connection peer's IP address and the transaction request. This applies to both an ordinary RPC service and Sentinel. The IP address is observed through the network connection; it is not a standard transaction field.

Evidence of Gas Station use can narrow down which service received the request. If the incident involved one of the submission paths examined here, MetaMask's Sentinel service would be a relevant place to look for connection records. This submission path explains the relevance of Kalland's request that MetaMask preserve off-chain logs.

## What the wallet sends to Sentinel

The source code shows two submission paths: Smart Transactions (STX) bundle submission and 7702 Relay submission. Both are Sentinel calling paths implemented by MetaMask.

| Submission path | Information sent to Sentinel | How the wallet address can be identified |
| --- | --- | --- |
| STX bundle submission | Signed transactions bundled together, including the user's intended transaction and transactions used to pay for gas. Submitted through the REST `submitTransactions` endpoint. [Submission code](https://github.com/MetaMask/core/blob/5c82ed964b68ede7cd6d1c045ddb593ac8bff437/packages/smart-transactions-controller/src/SmartTransactionsController.ts#L950-L988) | Recover the sender address from the signed transaction. |
| 7702 Relay submission | A signed delegation and execution data. Submitted through the JSON-RPC method `eth_sendRelayTransaction`. [Submission code](https://github.com/MetaMask/metamask-extension/blob/0997172ccbed2dfd31c56cf070b244d44cfb42ed/app/scripts/lib/transaction/hooks/delegation-7702-publish.ts#L158-L240) | Read the user's address from the delegation encoded in `data`. |

7702 Relay is MetaMask's relay submission mechanism that uses EIP-7702. Here, JSON-RPC describes the request and response format. The receiving service is Sentinel.

7702 Relay encodes a delegation containing the user's address into `data` and sends it. The following [delegation construction code](https://github.com/MetaMask/metamask-extension/blob/0997172ccbed2dfd31c56cf070b244d44cfb42ed/app/scripts/lib/transaction/delegation.ts#L298-L317) assigns the user's wallet address, `transaction.txParams.from`, to `delegator`:

```typescript
const unsignedDelegation: UnsignedDelegation = {
  delegator: transaction.txParams.from as Hex,
  delegate: delegatee ?? ANY_BENEFICIARY,
  authority: ROOT_AUTHORITY,
  salt,
  caveats,
};
```

After the delegation is signed and encoded, the address assigned to `delegator` is included in the submitted `data` alongside the execution data. Sentinel can decode this ABI-encoded data and read the user's address. If the server receiving the HTTPS connection also observes the peer IP, it could associate the wallet's request with that IP.

The wallet can communicate with a server before the user approves a transaction. When simulation is enabled and the support conditions are met, it can call `infura_simulateTransactions` while waiting for approval to obtain fee options for paying gas with tokens. This quote request includes the user's address in a `from` field and may also contain the destination and call data. Rejecting the transaction later does not undo a request that has already been sent.

The detailed diagram shows the default direct simulation route. With an active Shield subscription, the extension can instead route simulation requests through the Shield gateway. The [extension's configuration callback](https://github.com/MetaMask/metamask-extension/blob/0997172ccbed2dfd31c56cf070b244d44cfb42ed/app/scripts/wallet-init/instance-options/transaction-controller.ts#L272-L287) calls [gateway configuration code](https://github.com/MetaMask/metamask-extension/blob/0997172ccbed2dfd31c56cf070b244d44cfb42ed/shared/lib/shield/shield.ts#L17-L65) that can rewrite the request URL to `/proxy?url=...`. This routing exception was verified for simulation requests.

The submission path depends on the wallet version, network, and account type. STX and sendBundle support, as well as EIP-7702 support conditions, affect that choice. Other paths, such as Transaction Pay, also exist. Evidence of Gas Station use alone does not establish the exact API or wallet version used in the incident.

## Connecting an IP address to a particular transaction

A server receiving an HTTPS connection can observe its peer's IP address. A server terminating TLS can also read the decrypted request contents. If a wallet address is present in the request, or can be obtained from a signed transaction or delegation, the receiver could associate that address with the connection IP. Whether the IP is also passed to an internal application depends on the proxy configuration.

Tracing this relationship after the event requires retained records. For example, an access log could record the IP address, timestamp, and request identifier, while a transaction processing log separately records the wallet address and submission identifier or transaction hash. Linking the two requires an identifier or other evidence showing that both records refer to the same request.

Sentinel returns a UUID that the wallet uses to look up the submission's processing status. The wallet repeats the lookup while the status is `PENDING`. The following [result lookup code](https://github.com/MetaMask/metamask-extension/blob/0997172ccbed2dfd31c56cf070b244d44cfb42ed/app/scripts/lib/transaction/transaction-relay.ts#L58-L114) shows one lookup within that polling process:

```typescript
const { transactions } = await getSentinelApiMessenger().call(
  'SentinelApiService:getSmartTransaction',
  { chainId, uuid },
);

log('Polling response', transactions);

const transaction = transactions[0];

return {
  status: transaction?.status as string,
  transactionHash: transaction?.hash as Hex | undefined,
};
```

This code reads the status and hash of the first transaction returned by the UUID lookup. If `hash` is not yet available, `transactionHash` is `undefined`. When a hash is returned, it gives the wallet the on-chain transaction hash associated with the submission UUID.

The `log(...)` call writes the lookup response to a wallet-side debug log.

Using a UUID to trace a submission requires the server to retain the association between that UUID and the original request. If those records can also be linked to access logs, a known on-chain transaction hash could provide a route back to the submission request and its connection IP.

<figure id="relay-sequence">
<a href="/img/metamask-gas-station/7702-relay-sequence-revised.svg" target="_blank" rel="noopener">
<img src="/img/metamask-gas-station/7702-relay-sequence-revised.png" alt="7702 Relay submission and status lookup" width="1500" height="2165" style="height: auto;" loading="lazy" decoding="async">
</a>
<figcaption>7702 Relay submission and status lookup</figcaption>
</figure>

The solid horizontal arrows in the upper panel represent requests, and the dashed horizontal arrows represent responses. The lower panel shows on-chain submission conceptually; its placement does not establish its actual timing relative to the UUID response or status lookups. A transaction hash can exist before the transaction is included in a block.

For the STX path, retained signed transactions would allow their hashes to be calculated and compared with on-chain transactions.

If the backend retains access logs that can be linked to the transaction, there may be a way to identify the IP used for the request.

[TRM Labs](https://www.trmlabs.com/resources/blog/north-korea-steals-usd-6-36-billion-from-binance-in-the-largest-hack-in-history) attributed this incident to North Korea's TraderTraitor group.

Even if that attribution is correct, an attacker using a VPN or proxy could leave the intermediary's IP address in MetaMask's connection records instead of their original IP. In that case, connection records alone would be insufficient to establish the attacker's actual location or identity.

## Scope and limitations

MetaMask's wallet source code establishes which requests are sent to Sentinel and what transaction information they contain. It does not establish the server's log fields or retention practices.

The [MetaMask Privacy Policy](https://legal.consensys.io/metamask/privacy-policy/) describes temporary IP processing when needed to provide features or support security. The document alone does not establish whether Sentinel retains IP addresses linked to wallet addresses or UUIDs, or for how long.

This analysis uses the public Extension 13.56.0 source as of October 30, 2026, and the package versions pinned in its lockfile. The reference commit is `0997172`. I did not verify that the related packages and call paths in 13.55.0, the latest public release at the time of this research, match the analyzed source. The wallet version installed at the time of the incident was also not verified.

## References

- [Network-specific Sentinel API addresses](https://github.com/MetaMask/core/blob/b8d07a5a7731c565ad6c49b555b21b7cceedce61/packages/sentinel-api-service/src/constants.ts#L10-L89)
- [Submission path selection](https://github.com/MetaMask/metamask-extension/blob/0997172ccbed2dfd31c56cf070b244d44cfb42ed/app/scripts/lib/transaction/hooks/index.ts#L133-L215)
- [Signing delegations and encoding execution data](https://github.com/MetaMask/metamask-extension/blob/0997172ccbed2dfd31c56cf070b244d44cfb42ed/app/scripts/lib/transaction/delegation.ts#L192-L206)
- Pre-approval fee quotes: [request timing](https://github.com/MetaMask/core/blob/0bd8c26105066911fb73465c95c39ceea1206ee4/packages/transaction-controller/src/TransactionController.ts#L1301-L1338) and [request data](https://github.com/MetaMask/core/blob/0bd8c26105066911fb73465c95c39ceea1206ee4/packages/transaction-controller/src/utils/gas-fee-tokens.ts#L70-L123)
- [Ethereum transaction lifecycle](https://ethereum.org/developers/docs/transactions/#transaction-lifecycle)
- [Analyzed source snapshot](https://github.com/MetaMask/metamask-extension/tree/0997172ccbed2dfd31c56cf070b244d44cfb42ed): MetaMask Extension 13.56.0
