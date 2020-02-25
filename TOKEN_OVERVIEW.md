# Token Description

The [CACHE Gold Token (CGT)](https://cache.gold) is an ERC-20 compatible token in which 1 token represents 1 gram of pure gold. All gold tokens minted are backed by unencumbered gold held in reserves at participating fully audited vaults and may be redeemed for physical delivery or can be sold for fiat currency. Gold parcel tracking is performed through [GramChain](https://www.gramchain.net/) and all available bars currently locked and available for redemption can be view in the [GramChain Explorer](https://explorer.gramchain.net/).

While this token inherits the ERC-20 interface, there are extra properties of the token that are non-standard. In particular, the token has the ability to charge a storage fee that accrues over time as the token is held and a transfer fee that is paid when tokens are transferred from one address to another. These fees are paid in the native token itself ("in kind"), and are separate from the transaction fees paid in ETH when transacting on the Ethereum network. These fees are collected automatically whenever token holders transact.

### Quick Overview

* Symbol: `CGT`
* Name: `CACHE Gold`
* Decimals: `8`
* Mainnet Contract Address: [0xF5238462E7235c7B62811567E63Dd17d12C2EAA0](https://etherscan.io/token/0xF5238462E7235c7B62811567E63Dd17d12C2EAA0)
* Ropsten Contract Address: [0xa3872B300E97179E414D442b8Eef1862f641ED0e](https://ropsten.etherscan.io/token/0xa3872B300E97179E414D442b8Eef1862f641ED0e)
* Suggested gasLimit for `transfer`: 200,000
* Typical gas used for `trasfer`: 75,000 - 150,000

### Fee Overview

Please read the [Fees Guide](./FEES.md) for a detailed overview of the storage and transfer fees.

### Internal Addresses

There are 6 addresses associated with the CACHE Gold contract that have the special status as CACHE internal addresses. These addresses are not subject to storage or transfer fees.

1. [CACHE Owner](https://etherscan.io/address/0x3AB9c31148789570f51180a3eF7107E16c4B234c) : A Multi-Sig address owning the CACHE Gold Token contract and the only account able to mint tokens and perform other administrative functions, such as changing the transfer fee. This address is not meant to hold CACHE tokens.
2.  [Backed Treasury](https://etherscan.io/address/0x6522B05FE48d274f14559E0391BE3675E6A1AC91) : A Multi-Sig address where all tokens are minted and enter circulation. For tokens to be minted, an equivalent amount of gold must to be locked and unencumbered in a participating audited vault accounts, and the [Locked Gold Oracle](https://etherscan.io/address/0x5b7820E62778C7317403D892f6501DD816F82730) must be updated with this amount.
3.  [Unbacked Treasury](https://etherscan.io/address/0xD4033ea2eC53A26d6295f6f375D5C6afBe788660) : Address where tokens exit circulation. When tokens are redeemed for physical gold bars or unlocked from a vault, an equivalent amount of tokens will be moved to this address.
4.  [Fee Address](https://etherscan.io/address/0x7ea9b52e9f8673f3e22b4eec2c4c7a7e2d1b6636) : A Multi-Sig address where storage and transfer fees for external accounts are collected.
5.  [Storage Fee Enforcer](https://etherscan.io/address/0x8dCfA60A1fB2429a7AFc0266523F1BF7FBF42a1A) : An address authorized to force paying owed fees on inactive accounts. This address is not meant to hold CACHE tokens.
6.  [Redeem Address](https://etherscan.io/address/0xc8Bf2dBDe1d69d174fd40581f5177F684fA26eDa): The address tokens must be sent to when redeeming tokens for physical gold. CACHE uses a KYC process to whitelist accounts that send to this address and monitors deposits for redemption.

#### Transfer Restrictions

For security reasons, some addresses are restricted from sending to external addresses, and some can only receive tokens from certain addresses. The diagram below shows these rules. 

![Addresses](./img/addresses.png)

* The Redeem Address can only transfer to the Backed or Unbacked Treasury. This insures that if the private key is compromised CACHE tokens can only be moved to other Multi-Sig addresses controlled by CACHE.

* The Unbacked Treasury can only receive tokens from the Backed Treasury or the Redeem Address. When tokens enter the Unbacked Treasury the `totalCirculation()` decreases, therefore we want to ensure only addresses controlled by CACHE can transfer there. The Unbacked Treasury can only transfer tokens to the Backed Treasury, and it is only allowed when the Locked Gold Oracle has sufficient supply to allow these tokens to re-enter circulation.

### Minting and Locked Gold Oracle

The contract keeps track of the the real world supply of locked gold via the [Locked Gold Oracle](https://etherscan.io/address/0x5b7820E62778C7317403D892f6501DD816F82730) contract. It's only function is to keep track of the amount of gold currently locked in participating vaults as tracked by [GramChain](https://www.gramchain.net/), and the CACHE contract can check this value when adding new tokens to circulation to ensure it does not exceed the limit. 

The number of tokens in circulation at any given time is:
```
Total Supply - Unbacked Treasury
```
and it represented by the contract function [totalCirculation()](https://github.com/cache-token/cache-contract/blob/master/contracts/CacheGold.sol#L645-L647)

When more gold is locked in participating vaults and the Locked Gold Oracle value is increased, the contract owner can propose the [addBackedTokens()](https://github.com/cache-token/cache-contract/blob/master/contracts/CacheGold.sol#L268) function to increase the circulating supply by a given amount. The function will first move tokens from the Unbacked Treasury balance to the Backed Treasury, and then mint any additional tokens if the Unbacked balance in insufficient to cover the total amount of tokens to add. 

Similarly, if we want to reduce the circulating supply, tokens must be moved to the Unbacked Treasury (which is restricted from transferring tokens to external addresses), and then the Locked Gold Oracle can be reduced by an equivalent amount.

### Contract Events

All ERC-20 compatible tokens issue two native events:

* `event Transfer(from, to, value)`
* `event Approval(tokenOwner, spender, value);`

These allow any user to query ethereum nodes for a history of `Transfer` or `Approval` events, additionally being able to filter by the addresses either sending/receiving tokens or being approved for transfer.

In addition to these events, CACHE Tokens will emit four additional events:

* `event AddBackedTokens(amount)`
* `event RemoveTokens(amount)`

These represent events where the circulating supply of the token has changed, and can be used by a third party with auditing the history of the supply, or to listen for circulation changes.

* `event AccountInactive(address account, uint256 feePerYear)`
* `event AccountReActive(address account)`

These events are trigger when an account is marked inactive or reactivated. These events would allow anyone to quickly query which accounts are inactive based on the contract event log.
