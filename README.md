# Blur Exchange contest details
- $47,500 USDC main award pot
- $2,500 USDC gas optimization award pot
- Join [C4 Discord](https://discord.gg/code4rena) to register
- Submit findings [using the C4 form](https://code4rena.com/contests/2022-10-blur-exchange-contest/submit)
- [Read our guidelines for more details](https://docs.code4rena.com/roles/wardens)
- Starts October 5, 2022 20:00 UTC
- Ends October 8, 2022 20:00 UTC

# Blur Exchange

## Overview
The Blur Exchange is a single token exchange enabling transfers of ERC721/ERC1155 for ETH/WETH. It uses a ERC1967 proxy pattern and consists of three main components (1) [BlurExchange](./contracts/BlurExchange.sol), (2) [MatchingPolicy](./contracts/MatchingPolicy.sol), (3) [ExecutionDelegate](./contracts/ExecutionDelegate.sol).

### Architecture
![Exchange Architecture](./docs/exchange_architecture.png?raw=true)

### Signature Authentication

#### User Signatures
The exchange accepts two types of signature authentication determined by a `signatureVersion` parameter - single or bulk. Single listings are authenticated via a signature of the order hash.
  
##### Bulk Listing
To bulk list, the user will produce a merkle tree from the order hashes and sign the root. To verify, the respective merkle path for the order will be packed in `extraSignature`, the merkle root will be reconstructed from the order and merkle path, and the signature will be validated.


#### Oracle Signatures
This feature allows a user to opt-in to require an authorized oracle signature of the order with a recent block number. This enables an off-chain cancellation method where the oracle can continue to provide signatures to potential takers, until the user requests the oracle to stop. After some period of time, the old oracle signatures will expire.

To opt-in, the user has to set the `expirationTime` to 0. In order to fulfill the order, the oracle signature has to be packed in `extraSignature` and the `blockNumber` set to what was signed by the oracle.


### Order matching - [PolicyManager](./contracts/PolicyManager.sol)
In order to maintain flexibility with the types of orders and methods of matching that the exchange is able to execute, the order matching logic is separated to a set of whitelisted matching policies. The responsibility of each policy is to assert the criteria for a valid match are met and return the parameters for proper execution -
  - `price` - matching price
  - `tokenId` - NFT token id to transfer
  - `amount` - (for erc1155) amount of the token to transfer
  - `assetType` - `ERC721` or `ERC1155`


### Transfer approvals - [ExecutionDelegate](./contracts/ExecutionDelegate.sol)
Ultimately, token approval is only needed for calling transfer functions on `ERC721`, `ERC1155`, or `ERC20`. The `ExecutionDelegate` is a shared transfer proxy that can only call these transfer functions. There are additional safety features to ensure the proxy approval cannot be used maliciously.

#### Safety features
  - The calling contract must be approved on the `ExecutionDelegate`
  - Users have the ability to revoke approval from the `ExecutionDelegate` without having to individually calling every token contract.


### Cancellations
**On-chain methods**
  - `cancelOrder(Order order)` - must be called from `trader`; records order hash in `cancelledOrFilled` mapping that's checked when validating orders
  - `cancelOrders(Order[] orders)` - must be called from `trader`; calls `cancelOrder` for each order
  - `incrementNonce()` - increments the nonce of the `msg.sender`; all orders signed with the previous nonce are invalid

**Off-chain methods**
  - Oracle cancellations - if the order is signed with an `expirationTime` of 0, a user can request an oracle to stop producing authorization signatures; without a recent signature, the order will not be able to be matched


## Smart Contracts
All the contracts in this section are to be reviewed. Any contracts not in this list are to be ignored for this contest.


### BlurExchange.sol (359 sloc)
Core exchange contract responsible for coordinating the matching of orders and execution of the transfers.

It calls 3 external contracts
  - `PolicyManager`
  - `ExecutionDelegate`
  - Matching Policy

It uses 1 library
  - `MerkleVerifier`

It inherits the following contracts

#### EIP712.sol (134 sloc)
Contract containing all EIP712 compliant order hashing functions

#### ERC1967Proxy.sol (12 sloc)
Standard ERC1967 Proxy implementation

#### OrderStructs.sol (32 sloc)
Contains all necessary structs and enums for the Blur Exchange

#### ReentrancyGuarded.sol (10 sloc)
Modifier for reentrancy protection

#### MerkleVerifier.sol (38 sloc)
Library for Merkle tree computations

### ExecutionDelegate.sol (64 sloc)
Approved proxy to execute ERC721, ERC1155, and ERC20 transfers

Includes safety functions to allow for easy management of approvals by users

It calls 3 external contract interfaces
  - ERC721
  - ERC20
  - ERC1155

### PolicyManager.sol (42 sloc)
Contract reponsible for maintaining a whitelist for matching policies

### StandardPolicyERC721.sol (55 sloc)
Matching policy for standard fixed price sale of an ERC721 token

### StandardPolicyERC1155.sol (55 sloc)
Matching policy for standard fixed price sale of an ERC1155 token

## Development Documentation
Node version v16

First copy env - `cp .env.example .env`, then you can use the following commands:

- Install packages - `yarn`
- Compile contracts - `yarn compile`
- Test coverage - `yarn coverage`
- Run tests - `yarn test`
