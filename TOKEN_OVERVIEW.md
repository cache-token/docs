# Token Description

The Cache Gold Token is an ERC-20 compatible token in which 1 token represents 1 gram of gold. All gold tokens minted are backed by physical gold held in reserves at participating fully audited vaults.

While this token inherits the ERC-20 interface, there are extra properties of the token that are non-standard. In particular, the token has the ability to charge a storage fee that accrues over time as the token is held and a transfer fee that is paid when tokens are transferred to other unique address. These fees are totally separate from the transactions fees paid in ETH when interacting with smart contracts, and are instead paid in the native token itself. These fees are collected opportunistically when users transact.

### Fee Overview

Please read the [Fees Guide](./FEES.md)

### Internal Addresses

There are 6 addresses associated with the contract that have the special status as "internal" addresses. These addresses are not subject to storage or transfer fees.

1.  Owner Address : The address that deployed the contract and is the only account able to mint tokens. After the contract is deployed, this address will be changed to a MultiSig address so that tokens can only be minted pending approval of a group.
1.  Storage Fee Enforcer : This address is able to force paying storage fees or inactive fees on delinquent accounts. This is separate from the owner address, as we want a single private key that can force collection in a script without mulitisig interaction. 
2.  Backed Treasury Address : The backed treasury is where all tokens are minted to and where tokens enter circulation. For tokens to be minted, gold has to be locked in a participating audited vault. The Backed Treasury will be a MultiSig address controlled by multiple parties.
3.  Unbacked Treasury Address : The unbacked treasury is where tokens exit circulation. When gold bars need to be unlocked, the tokens will be moved here.
4.  Fee Address : The address where storage and transfer fees for external accounts are collected.
5.  Redeem Address: The address tokens must be sent to when redeeming tokens for physical gold. Cache will work in conjunction with a KYC process to whitelist addresses that send this address.

The Contract Owner has the ability to change any of these addresses if there is a risk they are compromised. 

### Minting and Locked Gold Oracle

The contract keeps touch with the real world supply of locked gold via the Locked Gold Oracle contract, which is very simple. It's only function is to keep track of the amount of gold currently locked in participating vaults, and the Cache contract can check this value when adding new tokens to circulation to ensure it does not exceed the limit. 

The number of tokens in circulation at any given time is:
```
The total supply - balance of unbacked treasury
```
and it represented by the contract function `totalCirculation()`

When more gold is locked in the vault and the Locked Gold Oracle value is increased, the contract owner will call `addBackedTokens()` function to increase the circulating supply by a given amount. The function will first move tokens from the Unbacked Treasury to the Backed Treasury, and then mint any additional tokens if the Unbacked balance in insufficient to cover the amount. 

Similarly, if we want to reduce the circulating supply, tokens must be moved to the Unbacked Treasury, and then the Locked Gold Oracle can be reduced by an equivalent amount.

#### Transfer Restrictions

For security reasons, and to follow token minting rules, some addresses are restricted from sending to external addresses, and some can only receive tokens from certain addresses. The diagram below shows these rules. 

![Addresses](./img/addresses.png)

The Redeem Address can only transfer to the Backed or Unbacked Treasury. This insures that if the private key is hacked, the hacker can only move the tokens to other addresses controlled by Cache.

The Unbacked Treasury can only receive tokens from the Backed Treasury or the Redeem Address. When tokens enter the Unbacked Treasury the `totalCirculation` decreases, therefore we want to ensure only addresses controlled by Cache can transfer there. The Unbacked Treasury can only transfer tokens to the Backed Treasury, and it is only allowed when the Locked Gold Oracle has sufficient supply to allow these tokens to re-enter circulation.

### Contract Events

All ERC-20 compatible tokens issue two native events:

* `event Transfer(from, to, value)`
* `event Approval(tokenOwner, spender, value);`

These allow any user to query ethereum nodes for a history of `Transfer` or `Approval` events, additionally being able to filter by the addresses either sending/receiving tokens or being approved for transfer.

In addition to these events, Cache Tokens will emit four additional events:

* `event AddBackedTokens(amount)`
* `event RemoveTokens(amount)`

These represent events where the circulating supply of the token has changed, and can be used by a third party with auditing the history of the supply, or to listen for circulation changes.

* `event AccountInactive(address account, uint256 feePerYear)`
* `event AccountReActive(address account)`

These events are trigger when an account is marked inactive or reactivated. These events are not completely necessary but will allow anyone to quickly query which accounts are inactive based on the contract event log.
