---
id: integration
title: Integration Guide for Ethererm Developers
---

import useBaseUrl from "@docusaurus/useBaseUrl";

import Link from "@docusaurus/Link";

Ethereum developers can simply use Godwoken as an EVM-compatible layer 2 chain, just like Arbitrum and Optimism.

This documentation can be used as an integration guide for wallets and exchanges, or as a reference for developing dApps on Godwoken. It describes the reason for developing dApps on Godwoken, the known caveats of version 1 and corresponding workarounds.

Prior knowledge of Ethereum is required for using this guide.

## Why Develop on Godwoken?

- **Security** - Nervos Network is a Proof-of-Work (PoW) layer 1 chain and Godwoken is an optimistic rollup layer 2 chain built on Nervos Network. The security model is quite different from other EVM-compatible PoS/DPoS/PoA chains or sidechains.
- **Low cost** - A typical Godwoken transaction currently costs less than $0.0001.
- **Ecosystem**
  - With Force Bridge, assets from 3 chains (Nervos, Ethereum and BSC) are already available on Godwoken. The support for more chains (Cardano, BTC) is on the way.
  - The TVL of Godwoken is now nearly [100 million](https://defillama.com/chains). A number of DeFi dApps (e.g. [YokaiSwap](https://www.yokaiswap.com/), [DARUMA](https://www.daruma.money/)) and wallets (e.g. [SafePal](https://www.safepal.io/download)) have already been integrated. 
- [**Interoperability 2.0**](https://medium.com/nervosnetwork/blockchain-abstraction-and-interoperability-2-0-eea98d81b7b6) - This may be the trump card of Godwoken. With the design abstractions of Nervos Network and Godwoken, it is possible to use any on-chain tool to port smart contracts to manipulate assets on the corresponding chain.

## Known Caveats

Godwoken V1 is still under development and targets 100% EVM compatibility. Having the best compatibility is the objective of designing and building Godwoken:

- The EVM being used in Godwoken must be 100% compatible with the latest forked version of Ethereum.
- Godwoken must be 100% compatible with Ethereum over the Web3 interfaces by using [Godwoken Web3](https://github.com/nervosnetwork/godwoken-web3).

Given the wide architectural and design differences between Godwoken and Ethereum, several discrepancies inevitably exist.

### Comparison with EVM

Godwoken targets 100% EVM compatibility and is designed to work with every smart contract that the latest Ethereum hard fork version supports. But, the current version is not yet fully compatible with EVM.

**EVM Revision**

The maximum EVM revision supported is `EVMC_BERLIN`.

#### pCKB

Godwoken v1 introduced a new concept, [**pCKB**](https://github.com/nervosnetwork/godwoken/blob/develop/docs/life_of_a_polyjuice_transaction.md#pckb) which is a defined layer 2 sUDT token type when deploying a Godwoken chain.

pCKB serves a similar purpose for the Godwoken chain as ETH does for the Ethereum chain, i.e., to collect transaction fees. In Ethereum, the gas for each smart contract is derived by calculation. And the transaction fee is then calculated by multiplying the gas with the specified gas price. In Godwoken, **pCKB** is the unit for calculating transaction fees. In other words, the gas price in Ethereum is calculated as ETH/gas (in wei, i.e. 10<sup>-18</sup> ETH), and the gas price in Godwoken is calculated as pCKB/gas. When Godwoken executes a transaction, it will deduct the transaction fee by using the layer 2 [sUDT](https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0025-simple-udt/0025-simple-udt.md) type, which is represented by **pCKB**.

Godwoken chain uses CKB as pCKB by default, while different Godwoken chains may use different token types as pCKB.

**Note**: With certain transactions being sent to the smart contract, the `value` of the transaction is `pCKB`.

#### Account Abstraction

Polyjuice only provides [contract accounts](https://ethereum.org/en/glossary/#contract-account). Godwoken's user accounts are leveraged as [EOAs](https://ethereum.org/en/glossary/#eoa).

#### All Tokens Are ERC20 Tokens

Ethereum processes ERC20 tokens differently from native ETH tokens, which is the reason wETH was invented. However, Godwoken conceals this difference.

All tokens on Godwoken are represented as Layer 2 sUDT types, regardless of whether they are native CKB or any sUDT types. Polyjuice proceeds from this Layer 2 sUDT [contract](https://github.com/nervosnetwork/godwoken-polyjuice/blob/b9c3ad4/solidity/erc20/SudtERC20Proxy_UserDefinedDecimals.sol) and ensures that all tokens on Godwoken are ERC20 compliant, regardless of whether supported by native CKB or sUDT. That is to say, it is unnecessary to distinguish between native tokens and ERC20 tokens. All the differenct tokens must be handled with the same ERC20 interface.

#### Transaction Structure

A Polyjuice transaction is essentially a Godwoken transaction. When Ethereum transactions are sent, they are converted to the Godwoken [RawL2Transaction](https://github.com/nervosnetwork/godwoken/blob/v1.0.0-rc1/crates/types/schemas/godwoken.mol#L69-L74) type when being sent, and are automatically processed by [Godwoken Web3](https://github.com/nervosnetwork/godwoken-web3/tree/v1.0.0-rc1).

#### Behavioral Differences of Some Opcodes

| EVM Opcode | Solidity Usage   | Behavior in Polyjuice         | Behavior in EVM                    |
| ---------- | ---------------- | ----------------------------- | ---------------------------------- |
| COINBASE   | block.coinbase   | address of the block_producer | address of the current block miner |
| GASLIMIT   | block.gaslimit   | 12,500,000                    | current block’s gas limit          |
| DIFFICULTY | block.difficulty | 2,500,000,000,000,000         | current block’s difficulty         |

#### Others

- Transaction context

  - `chain_id` is defined in Godwoken [RollupConfig#chain_id](https://github.com/nervosnetwork/godwoken/blob/a099f2010b212355f5504a8d464b6b70d29640a5/crates/types/schemas/godwoken.mol#L64).
  - the block difficulty is always `2500000000000000`
  - the gas limit  is 12500000 per block, but it is not a transaction-level limit. Any transaction can reach the gas limit
  - the size limit for contract's return data is `[25600B](https://github.com/nervosnetwork/godwoken-scripts/blob/31293d1/c/gw_def.h#L21-L22)`
  - the size limit for contract's storage is `[25600B](https://github.com/nervosnetwork/godwoken-scripts/blob/31293d1/c/gw_def.h#L21-L22)`

- `transaction.to` MUST be a contract address

  Direct transfer of the value (pCKB) from EOA to EOA is not supported.

  > Scenario: pCKB (CKB) is represented as an ERC20 token on layer2, which can be transferred through the sUDT_ERC20_Proxycontract transfer function.

- The `transfer value` can not exceed uint128:MAX

- Pre-compiled contract

  - `bn256_pairing` is not yet supported because of the high cycle cost (WIP)
  - [addition pre-compiled contracts](https://github.com/nervosnetwork/godwoken-polyjuice/blob/compatibility-breaking-changes/docs/Addition-Features.md)

### Godwoken Web3 API Compatibility

[Godwoken Web3 API](https://github.com/nervosnetwork/godwoken-web3) is a Web3 RPC compatible layer developed on top of Godwoken.

For more information about the usage, see [Ethereum RPC](https://eth.wiki/json-rpc/API) docs.

#### `transaction.to` MUST be a Contract Address

The `to` member of a Godwoken transaction must be a contract address.

**Result**

The `to` parameter of the following RPC methods must be a contract address, **not** an EOA address:

 - `eth_call`
 - `eth_estimateGas`
 - `eth_sendRawTransaction`

**Recommend Workaround**

Use the transfer function of the [CKB_ERC20_Proxy](https://github.com/nervosnetwork/godwoken-polyjuice/blob/3f1ad5b/solidity/erc20/README.md) contract [combined](https://github.com/nervosnetwork/godwoken-polyjuice/blob/3f1ad5b322/solidity/erc20/SudtERC20Proxy_UserDefinedDecimals.sol#L154) with `sudtId = 1` (CKB a.k.a. [pCKB](https://github.com/nervosnetwork/godwoken/blob/develop/docs/life_of_a_polyjuice_transaction.md#pckb)) to transfer assets from EOA To EOA.

----

#### Signing Transaction Only Support EIP155

Currently, we only support the [EIP155](https://eips.ethereum.org/EIPS/eip-155) signing scheme that incorporated `CHAIN_ID` for simple replay attack protection. 

**Result**

If you use an outdated Ethereum toolchain to send transactions, such as `truffle-hdwallet-provider`, you will experience failures.

**Recommend Workaround**

Make sure to use the latest Ethereum toolchain, such as [ether.js](https://docs.ethers.io/v5/) / [web3.js](https://web3js.readthedocs.io/en/v1.7.3/) / [truffle](https://trufflesuite.com/truffle/) / [@truffle/hdwallet-provider](https://github.com/trufflesuite/truffle-hdwallet-provider), etc.

----

#### Zero Address

Godwoken does not support the concept of [zero address](https://ethereum.org/ru/glossary/#zero-address) (0x0000000000000000000000000000000000000000). This means that Polyjuice cannot support the zero address as well.

**Result**

Transactions with the zero address in the `from`/`to` field are not supported.

**Recommend Workaround**

To use the zero address as a black hole to burn ethers, you can use the transfer function of the [CKB_ERC20_Proxy](https://github.com/nervosnetwork/godwoken-polyjuice/blob/3f1ad5b/solidity/erc20/README.md) contract to send ethers to the zero address.

For more information on the compatibility changes of Godwoken Web3 API, see [compatibility-breaking-changes](https://github.com/nervosnetwork/godwoken-web3/tree/compatibility-breaking-changes).

## Account Creation

It is mandatory to create an account on a Godwoken chain. Two ways to create a layer 2 account:
- Make a deposit to Godwoken at layer 1;
- Call the Godwoken built-in [meta_contract](https://github.com/nervosnetwork/godwoken-scripts/blob/86b299f/c/contracts/meta_contract.c) and create an account at layer 2.

## Hands-On EVM Training

This section covers Nervos' Layer 2 EVM training. With each task in this section, developers can practice step-by-step on developing EVM dapps on Nervos platform so as to gain direct development experience and prepare for deploying their own applications. 

### Prerequisites

- Connect MetaMask with Godwoken Testnet
- [Task Setup and Requirements - Layer 2 EVM (gitbook.io)](https://nervos.gitbook.io/layer-2-evm/task-setup-and-requirements)

### Tasks

The following nine tasks will walk developers through the process. Each task will give instructions on what needs to be done, the guidance on all the necessary tooling and informative materials needed for completing the tasks:

1. [Create account on the EVM Layer 2 Testnet](https://nervos.gitbook.io/layer-2-evm/tasks/1.create.godwoken.account)
2. [Deploy a Simple Ethereum Smart Contract on Polyjuice](https://nervos.gitbook.io/layer-2-evm/tasks/2.deploy.eth.contract)
3. [Issue a Smart Contract Call to the Deployed Smart Contract](https://nervos.gitbook.io/layer-2-evm/tasks/3.issue.contract.call)
4. [Issue an SUDT Token on Layer 1 and Deposit it to Layer 2](https://nervos.gitbook.io/layer-2-evm/tasks/4.issue.sudt.deposit)
5. [Deploy the ERC20 Proxy Contract for the Deposited SUDT](https://nervos.gitbook.io/layer-2-evm/tasks/5.deploy.erc20.proxy.contract)
6. [Use Force Bridge to Deposit Tokens on Ethereum to Polyjuice](https://nervos.gitbook.io/layer-2-evm/tasks/6.use.force.bridge.to.deposit)
7. [Port an Existing Ethereum dApp to Polyjuice](https://nervos.gitbook.io/layer-2-evm/tasks/7.port.eth.dapp)
8. [Modify the Ported dApp so it Supports Ethereum Assets via Force Bridge](https://nervos.gitbook.io/layer-2-evm/tasks/8.modify.dapp.support.force.bridge)
9. [Withdraw From Layer 2 Back to Layer 1](https://nervos.gitbook.io/layer-2-evm/tasks/9.withdraw)

