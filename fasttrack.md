# About this document

We prepared this document in the hopes that it will answer most of your questions regarding the matter of implementing native NFTs in the context of your application.

Since your core functionality is provided via Solidity contracts, we have taken care to provide you with as much examples and instructions as posible to help you navigate the transition from the existing ERC-721 tokens to the Unique native model.

Once you read theough the documentation, we propose that you attempt to replicate the follwing steps to master your challenge:
1. create a wallet selector that uses any of {polkadot.js wallet, Sublwallet, Talisman, Meatamask, Wallet Connect} wallet to connect to your app.
2. create a collecrtion using the SDK
3. mint a token using the SDK
4. write a Solidity contract in the EVM to use with the native NFT to change some NFT attribute, for example
5. call that contract from the SDK
6. nest a token

ALL the examples for this except point 6 are shown in this video: https://youtu.be/Cid_Ui5e0rk
To nest you just send a native token to a native token address as though it is a wallet. 

Read on...


# 1. evm and substrate

There are two types of accounts – substrate (5Grw...) and evm (0x...)
In terms of calling contracts:
- evm accounts operate the same way as in ethereum, nothing special
- Substrate accounts cannot call evm contracts directly because EVM works only with evm accounts. But they can do it through the `evm.call` extrinsic (in SDK - `sdk.evm.send` method). For contract, `msg.sender` will be substrate's account mirror – evm account. To calculate the substrate account mirror, use the `Address` utility from `@unique-nft/utils`

```ts
import { Address } from "@unique-nft/utils";

const ethMirror = Address.mirror.substrateToEthereum('5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY');
// 0xd43593c715Fdd31c61141ABd04a99FD6822c8558
```

or play with the converter in the documentation – https://docs.unique.network/reference/tools.html

It is essential to understand that mirror calculation is a one-way operation. You cannot calculate the origin address from its mirror.

```ts
import { Address } from "@unique-nft/utils";

const ethMirror = Address.mirror.substrateToEthereum('5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY');
// 0xd43593c715Fdd31c61141ABd04a99FD6822c8558

const subMirrorBack = Address.mirror.ethereumToSubstrate('0xd43593c715Fdd31c61141ABd04a99FD6822c8558');
// !!! different address 5FrLxJsyJ5x9n2rmxFwosFraxFCKcXZDngRLNectCn64UjtZ != 5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY
```

It is also worth noting that the evm mirror cannot be controlled directly (e.g., it cannot be added to MetaMask).

### `CrossAddress` struct

To make direct interaction between contracts and substrate accounts possible, we support the `CrossAddress` struct in Solidity. This struct represents the "ethereum or substrate" account. 

```Solidity
// Solidity
struct CrossAddress {
  address eth;
  uint256 sub;
}
```

*   For the EVM account, set the `eth` property with the EVM address (0x...), and the `sub` should be 0.
*   For the Substrate account, set the `sub` property with the substrate public key (not address!). The `eth` property should be equal to Ethereum zero address (0x000...00000);

To calculate substrate public key from an address in javascript, use `Address` utils.

```ts
import { Address } from "@unique-nft/utils";

const publicKey = Address.extract.substratePublicKey("5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY");
```

or convert the address to CrossAccount directly:

```ts
  const cross = Address.extract.ethCrossAccountId(
    "5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY",
    // or "0xd43593c715Fdd31c61141ABd04a99FD6822c8558"
  );
```

A lot of examples of how to use CrossAddress with evm and substrate accounts can be found in recipes of unique-contracts repo. For example:
- minter contract: https://github.com/UniqueNetwork/unique-contracts/blob/main/contracts/recipes/Minter.sol#L100
- test: how to call the minter contract with evm account https://github.com/UniqueNetwork/unique-contracts/blob/main/test/minter.spec.ts
- test: how to call the minter contract with substrate account (atm uses sdk 1.0): https://github.com/UniqueNetwork/unique-contracts/blob/main/test/minter.spec.ts

## Signers and Wallets

Documentation section on how to build applications, and specifically connecting accounts
https://docs.unique.network/build/sdk/v2/dapps.html#connecting-accounts

It could be much easier to understand playing with code. Here is a react template: https://github.com/UniqueNetwork/unique-react-template

## SDK

SDK is only for substrate accounts (remember that substrate accounts can invoke contracts).
The build section of the documentation explains all the needed concepts - https://docs.unique.network/build/sdk/v2/quick-start.html

## Why it makes sense to use the schema 2.0 and why you don't have to if you do not need it.

Unique Network implements NFTs natively – they are not contracts.

And yes, because of EVM support, plain ERC-721 NFTs can be created. However, no wallets, marketplaces, or other UIs in the ecosystem track this type of NFTs.

We support Unique Schema 2.0, an OpenSea compatible, on-chain metadata format, to make your metadata readable for all interfaces. So, you need to stick this format until you understand why not.

However, there is good news—if you use SDK or unique contracts, you don't need to understand this format in detail. You only need to understand its features. Everything else is handled for you.

The reference section in the documentation explaining all the features of unique schema – https://docs.unique.network/reference/schemas
The js library for the unique schema. You don't need it if you use SDK https://github.com/UniqueNetwork/unique_schemas 

## How to call EVM contracts using substrate account and SDK

docs – https://docs.unique.network/build/sdk/v2/evm.html

evm workshop – https://github.com/UniqueNetwork/unique-react-template/tree/workshop-evm
- [This function](https://github.com/UniqueNetwork/unique-react-template/blob/ab923457ece54f6ac6d1f2f47fc08ea52363dad1/src/pages/BreedingPage.tsx#L58-L107) covers how to invoke contracts 

Long story short:

- You save the contract's ABI as JSON file and import it
- You call sdk.evm.send and pass abi, contract address, function name, and params

## To change a schema 2.0 compliant data attribute from EVM

Use @unique-nft/contracts, [TokenManager.sol](https://github.com/UniqueNetwork/unique-contracts?tab=readme-ov-file#tokenmanagersol)

EVM workhop demonstrates how to do this.

- [How do we mutate token image](https://github.com/UniqueNetwork/unique-react-template/blob/ab923457ece54f6ac6d1f2f47fc08ea52363dad1/contracts/contracts/BreedingGame.sol#L111-L119) 
- [How do we mutate token attributes](https://github.com/UniqueNetwork/unique-react-template/blob/ab923457ece54f6ac6d1f2f47fc08ea52363dad1/contracts/contracts/BreedingGame.sol#L197-L202)
- [How do we call these solidity functions on the UI](https://github.com/UniqueNetwork/unique-react-template/blob/ab923457ece54f6ac6d1f2f47fc08ea52363dad1/src/pages/BreedingPage.tsx#L138-L173)

- Please remember to view this video:  https://youtu.be/Cid_Ui5e0rk
