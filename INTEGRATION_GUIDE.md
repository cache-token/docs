# Exchange Integration Guide

The Cache Gold Token (CGT) is an ERC-20 compatible token in which 1 token represents 1 gram of gold. It uses 8 decimal places and has symbol `CGT`.

While this token inherits the ERC-20 interface, there are extra properties of the token that may require additional work for exchange integration. 

In particular, there is:

* **A Transfer Fee** of 10 basis points (0.10%) on each transfer from one address to another. This fee is configurable and may be lowered in the future, but cannot rise higher than 0.1%.
* **A Storage Fee** of 25 basis points (0.25%) a year. This fee is not configurable.

Please fully read the [Fees Guide](./FEES.md) to fully understand the nature of these fees.

**MUST READ**: Because balances will naturally decay over time as storage fees accrue, it is essential to account for this. Any caching of an address' token balance in a database may eventually become out of sync with the real world balance. Additionally, transfer fees will be collected on each token transaction, so these fees must also be accounted for.

## Handling Transfer Fees

Each transfer of Cache tokens incurs a transfer fee of 10 basis points (0.10%), which is separate from the amount being sent. That is, if a user is holding 10 CGT and sends 5 CGT to the exchange, the exchange will receive 5 CGT and 5 * 0.001 = 0.005 GCT will be send to the Cache fee collection address, with a total of 5.005 tokens deducted from the user's wallet.

If the exchange then wants to transfers these 5 tokens to another address, like cold storage or liquidity pool, the maximum amount available to send is:

```
5 / 1.001 = 4.99500500

because

4.99500500 + (4.99500500 * 0.0001) [transfer fee] = 5 tokens
```

In fact, calling `balanceOf()` on the user's deposit address will return `4.99500500`, since it is the maximum sendable balance. Sending a transaction with the amount returned from `balanceOf()` should never fail, as it always accounts for owed storage fees and possible transfer fee on the next hop.

An exchange can expect that for a single user deposit, they may incur a single fee of 0.1% on the deposited amount if these funds are further moved to more secure storage or pooled with other deposits.

Similarly, the exchange can expect to incur a fee of 0.1% on amounts withdrawn to user addresses.

### Suggestions

* Limit unnecessary transfers, as each transfer will incur a 0.1% fee on the value of CGT sent.
* Transfer fees can be passed onto users via deposit and withdrawal fees. It may make sense for the fee to be proportional to the deposit or withdrawl amount, rather than flat.

### Basis Point Change

While the default transfer fee is 10 basis points, Cache has the ability to modify the transfer fee to anything between 0 and 10 basis points (whole numbers only). The current trasfer fee is readable via the `transferFeeBasisPoints()` contract function. 

In the event that the transfer fee is lowered in the future (which would be publicly announced), the exchange may want to also lower any fees passed onto the user.

## Handling Storage Fees

Cache token incurs a yearly storage fee of 25 basis points (0.25%), which is simply

```
storage fee owed =  balance * ( days since storage fees were last paid ) / 365.0 * 0.0025
```

The `balanceOf()` value will appear to decay over time for each day the token is held. In general the balance should
decay:

```
balance * (1 / 365.0) * 0.0025 = balance * 0.000006849 in fees per day
```

The contract also exposes a public function to calculate this directly:
```
storageFee(uint256 balance, uint daysSinceStoragePaid) returns(uint256)
```

### Grace Period

Cache had the ability to set a global grace period for storage fees. That is, a time period before storage fees begin accruing after an address first receives tokens. This is exposed via the public contract function
```
storageFeeGracePeriodDays() view returns(uint)
```

If a grace period is in effect, an exchange may chose to delay transferring a user deposit to cold storage until the grace period expires.

### Suggested Implementation

The user balance can decrease over time to account for storage fees the exchange is paying on behalf of the user.

The exchange can deduct storage fees before balance changes due to deposit, withdrawal, trades, etc. The logic can look something like the psuedo-code below:

```
function updateBalanceAndStorageFeeTimestamp(DBObject user, decimal balanceChange) {
    
    currentTimestamp = time.now();
    
    // initialize on first deposit
    if (user.lastPaidStorageFeeTimestamp is null) {
        user.lastPaidStorageFeeTimestamp = currentTimestamp;
    }

    // initialize balance to 0
    if (user.balance is null) {
        user.balance = 0
    }
    
    lastPaidStorageFeeTimestamp = user.lastPaidStorageFeeTimestamp;
    balance = user.balance;
    storageFee = 0;
    
    daysPassed = (currentTimestamp - lastPaidStorageFeeTimestamp).days;
    if (daysPassed > 0) {
        storageFee = currentBalance * daysPassed / 365.0 * 0.0025;
        // Update the time the last storage fee was paid in the DB
        user.lastPaidStorageFeeTimestamp = currentTimestamp;
    }

    // Update user balance in DB
    user.balance = balance - storageFee + balanceChange;
    user.save();
}
```

For example, Bob makes an initial deposit of 10 CGT.

Ten days later, he makes a market sell of 5 CGT, his new balance can become:
```
10 - (10 * 10/365.0 * 0.0025) - 5 = 4.999315068
```

If he then deposits another 5 tokens, 15 days later, his balance will become:
```
4.999315068 - (4.999315068 * 15/365.0 * 0.0025) + 5 = 9.99880144
```

Note that the above function can also be called even if there is no balance change (passing 0 as `balanceChange`), just to deduct the accumulated storage fees, and update the user balance. 

User CGT balances could be updated to account for accrued fees:

1. When a user logs in
2. Before being able to post a sell order
3. Before being able to perform a withdrawal

### Avoid Order Insolvency

Note that because of the balance decay, it is possible for a user's balance to dip below the amount of a previously open sell order. In this event, the order should fail to fully fill, however this may cause obvious user frustration.

To avoid this condition, it may make sense to only allow users to post CGT orders that are some small percentage below their current balance, and have the orders time out after this period.

For instance, if the user is only allowed to post a sell order for 99.9% of their token balance, the .1% left would be able to pay for 146 days worth of storage fees (since .001/.0025 * 365 = 146). If the order still remained open for 146 days, the exchange could either automatically cancel the order, or reduce it's amount further.

## Avoiding inactive fees

If an address holding CGT is inactive for more than 3 years (has not created a successful transaction interacting with the Cache contract), the account can be marked inactive, and fees are increased. This is unlikely to happen on an exchange with many active addresses, however if the exchange has any address that may hold CGT with no transactions for an extended period of time, they should be aware to transact on this address at least every 1094 days.

Suggested transactions to signal the account is active:

1. `approve()` 0 CGT to self
2. `transfer()` 0 CGT to self
3. `payStorageFee()` 

## Estimated Gas Costs

* `transfer` : is ~130,000 on first transfer and ~75,000 on subsequent transfers
* `approve` : is ~66,000 on first approve and ~37,000 on subsequent approves
* `transferFrom` : ~162,000 on first transferFrom and ~83000 on subsequent calls 
* `payStorageFee` : ~51,000 

## Advanced Contract Interaction

The contract exposes several functions to be able to pull detailed information about owed fees, addresses activity, days since storage fees have been paid, etc. 

* `balanceOf(address owner) view returns (uint256)` : Returns the current balance on the account including owed fees. This is synonamous with the "the maximum amount an account could send to another address".
* `balanceOfNoFees(address owner) view returns (uint256)` : Returns the balance stored in contract storage, not accounting for any owed fees.
* `daysSincePaidStorageFee(address account) view returns(uint)` : Will return how many days since an address has last paid storage fees
* `daysSinceActivity(address account) view returns(uint)` : Will return how many days since the address last originated a transaction to the contract
* `calcOwedFees(address account) view returns(uint256)` : Will return the current fees owed on the address
* `calcTransferFee(address account, uint256 value) view returns(uint256)` : Will return the expected transfer fee on `account` sending `value` tokens
* `storageFee(uint256 balance, uint daysSinceStoragePaid) pure returns(uint256)` : Will return the expected storage fee on `balance` for `daysSinceStoragePaid` days
* `function storageFeeGracePeriodDays() view returns(uint)` : The current storage fee grace period for addresses receiving tokens for the first time
* `function transferFeeBasisPoints() view returns(uint)` : The current transfer fee in basis points. Value will be an integer between [0, 10] inclusive.
