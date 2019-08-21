# Understanding Cache Gold Token Fees

## Storage Fee

The storage fee is meant to mimic the standard fees paid when storing physical gold in secured vaults. The fee is structured on a per annum basis at 25 basis points (0.25%) and is not configurable.

#### How to Pay Storage Fees

The storage fee can be paid in three ways: 

1.  Transfering tokens to another address.
2.  Transfering any amount of tokens to same address (i.e. the sender and receiver are identical).
3.  Making a transaction to call the contract's `payStorageFee()` function.

When sending tokens to another address via the ERC-20 `transfer()` function, any owed storage fees will automatically be collected. **NOTE** If the receiving address has a storage fee balance due, the storage fee will automatically be deducted from this address as well (both sender and receiver pay). In this way, storage fees are opportunistically collected whenever a user regularly transacts. For most cases, users do not have to worry about paying their fees, it will happen naturally.

If a user is planning to hold their tokens for long periods of time, they may also pay storage fees by sending 0 or more tokens back to their own address (send to self). There is no transfer fee when transferring to the same address. This will be the easiest way for _most_ users to manually pay storage fees.

Lastly, for saavy users and exchanges, storage fees can also be paid by constructing a transaction to call the contract function `payStorageFee()`. This operation is less complex (less EVM operations) than doing a transfer to the same address, and therefore will require less Gas to execute.

Each time a storage fee is paid, either explicitly or via transfer, the storage fee clock on the account is reset to 0 days. For contract purposes, 1 day is exactly 86,400 seconds, and a year is exactly 365 days.

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
The contract can set a configurable grace period before storage fees begin to accrue on an account. The current grace period is avaliable via the contract function `storageFeeGracePeriodDays()`. The grace period is stored per address, so that if the global grace period changes while in effect for an address, it's grace period can be retroactively honored. The grace period should only be valid once per address. That is, if an account's grace period has expired, the entire token balance is sent to another address, and then more tokens are received, the grace period will not reset.

#### Force Paying Fees
If it has been more than 365 days since storage fees were last paid on a particular address, the contract owner has the option to force collecting owed storage fees on delinquent accounts.

#### Inactive Fee
The contract also implements an inactive fee. If an address has not interacted with the contract (originated a transaction to the contract) for 3 or more years, the account may be flagged as "inactive". When it is marked inactive, the owed storage fees are deducted and the balance after storage fees is taken as a snapshot. A yearly inactive fee of 50 basis points (0.5%) of the snapshot balance or 1 token, whichever is greater, is set on the account. 

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

If at any point a user owning an address resumes any activity (originates a transaction to the contract), the account is "reactivated", previously owed fees are deducted, and the storage fee clock is reset to 0. The account cannot be marked inactive for another 3 years of inactivity.

**Inactive fees are totally disjoint and non-coincident with storage fees**. When an account is marked inactive, previously owed storage fees are paid automatically, and inactive fees begin accruing from the moment the account had been inactive for 3 years.

An account can be marked inactive in three ways.

1. The contract owner can make a transaction to mark it inactive after 3 or more years of inactivity
2. If an account receives tokens after having been inactive for a long period, it is marked inactive. That is, we opportunistically mark it inactive when another account sends tokens to that address.
3. If an account has been inactive for more than 3 years, but was never marked inactive by the contract owner, and then the account makes a transaction after a long period of dormancy (found lost keys), transaction will temporarily the account to inactive, pay all owed fees, and then unmark it as inactive. 

Therefore in all cases, if an account is ever inactive for more than 3 years, it wil always pay all owed storage and inactive fees. Also note if the contract owner forces paying storage fees on an overdue account it does not alter the 'inactivity' clock for that account. Therefore, the contract owner can continue collecting owed storage fees on accounts and still mark it as inactive when it later crosses the 3 year inactivity threshold.

Users can avoid having inactive accounts by simply making any valid transaction with the token (like sending to self) at least every 1094 days. 

## Transfer fee
The transfer fee is a simple mechanism that charges 10 basis points (0.10%) on the amount of tokens sent in each `transfer`. The transfer fee will not deduct from the total sent, but will be in addition to the sending amount. For instance, if 5 tokens are sent from Alice to Bob, the total transfer amount will be 5.005, with 5 tokens going to Bob and 0.005 tokens being sent to a fee collection address controlled by Cache.

Note that transferring tokens to the same address incurs no transfer fee. This as a simple case to pay storage fees without calling a special contract function. 

## Example Transactions With Fees

#### Case 1
**Action** : Alice has 10 tokens held for 30 days (without paying storage fees). She wants to send 5 tokens to Bob, who currently has no tokens. Alice makes a simple ERC20 transfer sending 5 tokens to Bob.

**Result** : Alice will have a final balance of:
```
10 - 5 - (10 * 30.0/365.0 * 0.0025) [storage fee] - (5 * 0.001) [transfer fee] = 4.99294520
```

Bob will have a final balance of: 
```
5
```

Bob pays no storage fee because he's never had tokens before and is not subject to transfer fees because he is not the sender. 

#### Case 2
**Action** : Alice has 10 tokens held for 30 days. She wants to send 5 tokens to Bob, who currently has had 1 token for 45 days. Alice makes a simple ERC20 transfer sending 5 tokens to Bob.

**Result** : Alice will have a final balance of:
```
10 - 5 - (10 * 30.0/365.0 * 0.0025) [storage fee] - (5 * 0.001) [transfer fee] = 4.99294520
```

Bob will have a final balance of:
```
10 + 1 - (1 * 45.0/365.0 * 0.0025) [storage fee] = 10.99969178
```
In this case Bob will pay a storage fee on the tokens he already had, before resetting the clock on his new balance.

#### Case 3
**Action** : Alice has 10 tokens held for 30 days and wants to pay a storage fee. She makes a token transfer of any amount back to her same address.

**Result** : Alice will have a final balance of:
```
10 - (10 * 30.0/365.0 * 0.0025) [storage fee] = 9.99794520
```

There are no transfer fees when sending to the same address.

## Important Note On Balance Representation

The Cache contract is unique in that it will show the user balance `balanceOf` not as it is currently stored in the contract storage, but the balance taking into account owed fees. This is a deliberate design choice made to allow compatibility with existing wallets so that choosing to "Send Entire Balance" as shown by `balanceOf`, will not fail. This contract implements an additional function `balanceOfNoFees`, which is the typical ERC-20 implementation that shows the contract storage value of the current balance, without taking owed fees into consideration. 

Because of the storage fee, the `balanceOf` an address will appear to decay over time as fees accrue. One aspect of this design is that users may expect to have already paid their storage fee, when in fact, it has not actually been collected yet. However, when the user does make a transaction with the token, the owed storage fees will be automatically collected.

The transfer fee also adds another modification to the shown balance. For our design, we desire for transactions to not deduct the transfer fee from the amount sent, while also desiring for the 'Send Entire Balance' on existing wallets to still work. In order to acheive this goal, the `balanceOf` functions shows the "Maximum you could send, taking into account the transfer fee". For instance, if there is no storage fee due, and a user currently has a balance of 10 tokens in contract storage, and the transfer fee is 10 basis points, the maximum they could send is:
```
10/1.001 = 9.99000999
```
Because sending 9.99000999 incurs a fee of 0.00999001, and 
```
9.99000999 + 0.00999001 = 10, the amount of tokens held
```
So `balanceOf()` will show `9.99000999` instead of `10.00000000`

In this sense, the balance shown is already 'paying for the next hop' in transfer fees. For a postal methapor, it's like sending a letter with a stamp included for return. While the design seems more complicated, it solves several issues. If `balanceOf` didn't take owed fees into account, the contract would entercounter situations where it is unable to transfer the amount requested after fees are deducted, leading to non-intuitive transaction failures. In our design, the user will always be able to send what is shown in `balanceOf` and the receiver will actually receive the amount sent, not the (amount - deducted fees).
