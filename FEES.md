# Understanding CACHE Gold Token Fees

## Storage Fee

The storage fee is meant to mimic the standard fees paid when storing physical gold in vaults. The fee is structured on a per annum basis at 25 basis points (0.25%) and is not configurable.

#### How Storage Fees are Paid

The storage fee can be paid in three ways: 

1.  Transfering tokens to another address.
2.  Transfering any amount of tokens to same address (i.e. the sender and receiver address are identical).
3.  Making a transaction to call the contract's [payStorageFee()](https://github.com/cache-token/cache-contract/blob/master/contracts/CacheGold.sol#L304) function.

When sending tokens to another address via the ERC-20 `transfer()` function, any accrued storage fees will automatically be collected. **NOTE:** If the receiving address has a storage fee balance due, the storage fee will automatically be deducted from this address as well (both sender and receiver pay). In this way, storage fees are automatically collected whenever a token holder transacts. For most cases, token holders do not have to worry about paying their fees, it will happen automatically whenever they send or receive CACHE Gold Tokens.

If a user is planning to hold their tokens in an address for a long period of time, they may also pay storage fees by sending 0 or more tokens back to their own address (send to self). There is no transfer fee when transferring to the same address. This will be the easiest way for _most_ users to manually pay storage fees.

Lastly, for saavy users and exchanges, storage fees can also be paid by constructing a transaction to call the contract function `payStorageFee()`. This operation is less complex (less EVM operations) than doing a transfer to the same address, and therefore will require less gas to execute.

Each time a storage fee is paid, either explicitly or via transfer, the storage fee clock on the account is reset to 0 days. For contract purposes, 1 day is exactly 86,400 seconds, and one year is exactly 365 days.

#### Storage Fee Calculation

The current storage fees are 25 basis points per annum (0.25%) 

An example JavaScript function implementing this fee structure:

```javascript
function calcStorageFee(balance, days_passed) {
    let fee = days_passed / 365.0 * 0.0025;
    if (balance < fee)
        return balance;
    }
    return fee;
}
```

#### Storage Fee Grace Period
The contract can set a configurable grace period before storage fees begin to accrue on an account. The current grace period is avaliable via the contract function `storageFeeGracePeriodDays()`. The grace period is stored per address, so that if the global grace period changes while in effect for an address, it's grace period can be retroactively honored. The grace period should only be valid once per address. That is, if an account's grace period has expired, receiving more tokens on the address will not restart the grace period.

#### Force Collecting Fees
If it has been more than 365 days since storage fees were last paid on a particular address, the contract owner has the option to force collecting accrued storage fees on these inactive accounts.

#### Inactive Fee
The contract also implements an inactive fee. If an address has not interacted with the contract (originated a transaction to the contract) for 3 or more years, the account may be flagged as "inactive". When it is marked inactive, the owed storage fees are deducted and the balance after deducting storage fees is taken as a snapshot. A yearly inactive fee of 50 basis points (0.5%) of the snapshot balance or 1 token, whichever is greater, is set on the account. 

For example, if an account has been inactive for 3 years, and has a balance of 1000 tokens, without ever having paid storage fees.
```
It would owe 1000 * 0.0025 * 3 = 7.5 tokens in storage fees.
After this is deducted the remaining balance at 3 years is: 1000 - 7.5 = 992.5 which becomes the snapshot balance
The yearly inactive fee on 992.5 tokens = 992.5 * 0.005 = 4.9625 tokens
```

Similarly, if the account has been inactive for 3 years, and only had a balance of 5 tokens.
```
It would owe 5 * 0.0025 * 3 = 0.0375 tokens in storage fees.
After this is deducted the remaining balance at 3 years is: 5 - 0.0375 = 4.9625, which becomes the snapshot balance
The yearly inactive fee on a snapshot balance of 4.9625 tokens is 1 token per year, as 1 token > 50 basis points on 4.9625 tokens.
```

The contract owner has the ability to force collection of these inactive fees on a prorated basis at any point in time on inactive accounts.

If at any point an account resumes any activity (originates a transaction to the contract), the account is "reactivated", previously owed fees are deducted, and the storage fee clock is reset to 0. The account cannot be marked inactive unless it reamains inactive for another 3 years from the reset date.

**Inactive fees are totally distinct from and non-coincident with storage fees**. When an account is marked inactive, previously owed storage fees are paid automatically, and inactive fees begin accruing from the moment the account had been inactive for 3 years.

An account can be marked inactive in three ways.

1. The contract owner can make a transaction to mark it inactive after 3 or more years of inactivity.
2. If an account receives tokens after having been inactive for a long period, it is marked inactive. That is, we automatically mark an account inactive when another account sends tokens to that address.
3. If an account has been inactive for 3 years, but was never marked inactive by the contract owner, and then the account makes a transaction after a long period of dormancy (like when recovering lost keys), the transaction will temporarily set the account to inactive, pay all owed fees, and then unmark it as inactive. 

Therefore in all cases, if an account is ever inactive for 3 years or more, it wil always pay all owed storage and inactive fees. Also note if the contract owner forces paying storage fees on an overdue account it does not alter the 'inactivity' clock for that account. Therefore, the contract owner can continue collecting owed storage fees on accounts and still mark it as inactive when it later crosses the 3 year inactivity threshold.

Users can avoid having inactive accounts by simply making any valid transaction with the token (such as sending to self) at least once every 1094 days. 

## Transfer fee
The transfer fee is a simple mechanism that charges 10 basis points (0.10%) on the amount of tokens sent in each `transfer`. The transfer fee will not deduct from the total sent, but will be in addition to the sending amount. For instance, if 5 tokens are sent from Alice to Bob, the total transfer amount will be 5.005, with 5 tokens going to Bob and 0.005 tokens being sent to a fee collection address controlled by Cache.

Note that transferring tokens to the same address incurs no transfer fee. This is a simple way to pay storage fees without calling a special contract function. 

## Important Note On Balance Representation

### Balance Decay
The CACHE contract is unique in that it will show the user balance `balanceOf()` not as it is currently stored in the contract storage, but the balance taking into account owed fees. Token balances appear to decay over time, even if the accrued fees have not yet been collected. This is a deliberate design choice to allow compatibility with existing wallets so that choosing to "Send Entire Balance" or "Maximum" as shown by `balanceOf()`, will not fail due to insufficient balance after fees. The contract implements an additional function `balanceOfNoFees()`, which is the typical ERC-20 implementation that shows the contract storage value of the current balance, without taking owed fees into consideration.

### Transfer Fee Included
The transfer fee also adds another modification to the shown balance. For our design, we desire for transactions to not deduct the transfer fee from the amount sent, while also desiring for the 'Send Entire Balance' on existing wallets to still work. In order to acheive this goal, the `balanceOf` functions shows the maximum amount that can be sent, taking into account the transfer fee. For instance if a user, Alice, received 10 tokens, has no storage fees due, and the transfer fee is 10 basis points, the maximum she could send is:
```
10/1.001 = 9.99000999
```
Because sending 9.99000999 incurs a fee of 0.00999001, and 
```
9.99000999 + 0.00999001 = 10, the amount received
```
So `balanceOf()` will show `9.99000999` instead of `10.00000000` and `balanceOfNoFees()` would show the expected `10.00000000` (the amount received).

In this sense, the balance shown is already 'paying for the next hop' in transfer fees. If Alice then sent `9.99000999` tokens to Bob, a `Transfer` event would be emitted would show the amount sent as `9.99000999`. If Bob then checked his `balanceOf()`, it would show:
```
9.99000999 / 1.001 = 9.98002996
```
and `balanceOfNoFees()` would simply show `9.99000999`.

In our design, the user will always be able to send what is shown in `balanceOf` and the receiver can monitor for a `Transfer` event for that full amount, not the (amount - transfer fees).

## Example Transactions With Fees and Balances

### Case 1
**Action** : Alice has 10 tokens held for 30 days (without paying storage fees). She wants to send 5 tokens to Bob, who currently has no tokens. Alice makes a simple ERC20 transfer sending 5 tokens to Bob.

**Result** : After the transfer

`balanceOfNoFees(Alice)` would return :
```
10 - 5 - (10 * 30.0/365.0 * 0.0025) [storage fee] - (5 * 0.001) [transfer fee] = 4.99294521
```
and calling `balanceOf(Alice)` would return :
```
4.99294521 / 1.001 = 4.98795726
```
which accounts for transfer fees on the next hop if sending entire balance

`balanceOfNoFees(Bob)` would return :
```
5.00000000
```
and `balanceOf(Bob)` would return :
```
5 / 1.001 = 4.99500500
```
Bob pays no storage fee because he's never had tokens before and is not subject to transfer fees because he is not the sender. 

The transaction would emit two `Transfer` events:
1. From Alice -> Bob for `5` tokens
2. From Alice -> Cache Fee Address for `0.00705479` tokens, which accounts for Alice's accrued storage fee and the transfer fee on `5` tokens

### Case 2
**Action** : Alice has 10 tokens held for 30 days. She wants to send 5 tokens to Bob, who currently has had 1 token for 45 days. Alice makes a simple ERC-20 transfer sending 5 tokens to Bob.

**Result** : After the transfer

`balanceOfNoFees(Alice)` would return :
```
10 - 5 - (10 * 30.0/365.0 * 0.0025) [storage fee] - (5 * 0.001) [transfer fee] = 4.99294521
```
and calling `balanceOf(Alice)` would return :
```
4.99294521 / 1.001 = 4.98795726
```

`balanceOfNoFees(Bob)` would return :
```
5 + 1 - (1 * 45.0/365.0 * 0.0025) [storage fee] = 5.99969179
```
and calling `balanceOf(Bob)` would return :
```
5.99969179 / 1.001 = 5.99369810
```
In this case Bob will pay a storage fee on the tokens he already had, before resetting the clock on his new balance.

The transaction would emit three `Transfer` events:
1. From Alice -> Bob for `5` tokens
2. From Alice -> Cache Fee Address for `0.00705479` tokens, which accounts for Alice's storage fee and the transfer fee on `5` tokens
3. From Bob -> Cache Fee Address for `0.00030821` tokens, for Bob's storage fee on 1 token for 45 days

### Case 3
**Action** : Alice has 10 tokens held for 30 days and wants to pay a storage fee. She makes a transfer of 0 tokens back to herself.

**Result** : After the transfer
`balanceOfNoFees(Alice)` would return :
```
10 - (10 * 30.0/365.0 * 0.0025) [storage fee] = 9.99794521
```
`balanceOf(Alice)` would return : 
```
9.99794521 / 1.001 = 9.98795726
```
There are no transfer fees when sending to the same address.

This transfer would emit 2 events:
1. Alice -> Alice for `0` tokens
2. Alice -> Cache Fee Address for `0.00205479` tokens, the storage fees owed
