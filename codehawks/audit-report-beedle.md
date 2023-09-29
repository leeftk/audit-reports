# **[H-01]** - Precision loss leads to inaccurate pool values

## Summary

The `_calculateInterest` function is vulnerable to precision loss. Due to dividing twice the function will return zero for both interest and `protocolInterest`. This could lead to a lender giving their loan to another pool that doesn't have the balance to cover it. Leading to a loss for them and a gain for the future pools as they will have debts greater than their balance.

## Vulnerability Details

`giveLoans()` calls `_calculateInterest()` which then returns the amount of interest the protocol fee and adds that to the loan (debt) in order to calculate the `totalDebt`. 
However, we already determined that these values are vulnerable to precision loss leading them to return 0 in some cases or be smaller than expected. That would lead to an inaccurate `totalDebt` allowing us to bypass the check in `giveLoans` that requires `poolBalance` to be greater than `totalDebt`.

```solidity

            uint256 totalDebt = loan.debt + lenderInterest + protocolInterest;
            if (pool.poolBalance < totalDebt) revert PoolTooSmall();


```
## Impact

Alice creates a pool with 100 dollars in it.

Let's say Bob takes out a loan for 100 dollars in debt.

This loan has a five percent interest rate leaving Bob owing a total of 105 after a year.

Alice decides she wants to get rid of the loan so she looks for another pool to give it to.

In theory, the protocol should only allow Alice to give the loan to a pool that has greater then a 105 dollar balance.

However when calling `giveloans` the function will call `calculateInterest` to calculate the interest to be 0 and assume `totalDebt` is 100. This would allow Alice to give her loan to a pool that has 101 or more dollars in it which isn't the intended behavior. The effect here would be that the user would be able to get an interest-free loan and the `outstandingLoans` value would be incorrect. Alice of course would lose the interest and now you have a loans that have bad debt or miscalculated values in `outstandingLoans` breaking trust in the entire system. This is also true of the `buyLoans` function `refinanceFunction` and any other function that calls `_calculateInterest` in order to calculate the `totalDebt`. There are multiple values throughout the contract that depend on `totalDebt` to be accurate as you can see below from the `giveLoan` function.

```
            _updatePoolBalance(poolId, pool.poolBalance - totalDebt);
            pools[poolId].outstandingLoans += totalDebt;
```


## Recommended Steps

Verifying that neither the interest rate nor the fees are zero can help prevent precision loss and trigger a reversal if necessary. Additionally, establishing a minimum loan size could ensure that the left side of the equation consistently exceeds the right side.

```
 interest = (l.interestRate * l.debt * timeElapsed) / 10000 / 365 days;

        fees = (lenderFee * interest) / 10000;
        if(interest == 0 || fees == 0) revert PrecisionLoss();
        interest -= fees;

```

Another solution is creating a formula that always rounds up. This is in favor of the lender and the protocol as well. Something like this would suffice.

```
    /**
     * @param a numerator
     * @param b denominator
     * @dev Division, rounded up
     */
    function roundUpDiv(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) return 0;
        return (a - 1) / b + 1;
    }
```

# **[H-02]** - Fee on transfer tokens can cause lender tokens to get stuck in contract.

## Summary

Fee on transfer tokens take a tax when they are submitted

## Vulnerability Details

Some ERC20 tokens, such as USDT, allow for charging a fee any time transfer() or transferFrom() is called. If a contract does not allow for amounts to change after transfers, subsequent transfer operations based on the original amount will revert() due to the contract having an insufficient balance. However, a malicious user could also take advantage of this to steal funds from the pool.

Let's take a look at the `addPool()` and `removePool()` functions. 

```
    /// @notice remove from the pool balance
    /// can only be called by the pool lender
    /// @param poolId the id of the pool to remove from
    /// @param amount the amount to remove
    function removeFromPool(bytes32 poolId, uint256 amount) external {
        if (pools[poolId].lender != msg.sender) revert Unauthorized();
        if (amount == 0) revert PoolConfig();
        _updatePoolBalance(poolId, pools[poolId].poolBalance - amount);
        // transfer the loan tokens from the contract to the lender
        IERC20(pools[poolId].loanToken).transfer(msg.sender, amount);
    }

```

```
    /// @notice add to the pool balance
    /// can only be called by the pool lender
    /// @param poolId the id of the pool to add to
    /// @param amount the amount to add
    function addToPool(bytes32 poolId, uint256 amount) external {
        if (pools[poolId].lender != msg.sender) revert Unauthorized();
        if (amount == 0) revert PoolConfig();

        _updatePoolBalance(poolId, pools[poolId].poolBalance + amount);
        // transfer the loan tokens from the lender to the contract
        IERC20(pools[poolId].loanToken).transferFrom(
            msg.sender,
            address(this),
            amount
        );
    }

```
## Impact

`addToPool()` takes in an amount that is set by the users. `removeFromPool()` does the same, however when attempting to remove a pool using fee on transfer tokens the function would revert due to insufficient balance leaving the users funds stuck in the pool. Consider the example below.

Step to Reproduce

1. Alice sends 100 of FEE tokens to the contract when calling `addToPool()`.
2. FEE token contract takes 10% of the tokens (10 FEE).
3. 90 FEE tokens actually get deposited in contract.
4. `_updatePoolBalance(poolId, pools[poolId].poolBalance + amount);` will equal 100.
5. Attacker then sends 100 FEE tokens to the contract
6. The contract now has 180 FEE tokens but each user has an accounting of 100 FEE.
6. The attacker then tries to redeem his collateral for the full amount 100 FEE tokens.
7. The contract will transfer 100 FEE tokens to Bob taking 10 of Alices tokens with him.
8. Bob can then deposit back into the pool and repeat this until he drains all of Alice's funds.
9. When Alice attempts to withdraw the transaction will revert due to insufficient funds.

## Tools used

Manual Review

## Recommended Steps

Measure the contract balance before and after the call to transfer()/transferFrom() and use the difference between the two as the amount, rather than the amount stated

```
   function addToPool(bytes32 poolId, uint256 amount) external {
        if (pools[poolId].lender != msg.sender) revert Unauthorized();
        if (amount == 0) revert PoolConfig();
        uint256 balanceBefore = IERC20(pools[poolId].loanToken).balanceOf(address(this))

        IERC20(pools[poolId].loanToken).transferFrom(
               msg.sender,
               address(this),
               amount
           );
        uint256 balanceAfter = IERC20(pools[poolId].loanToken).balanceOf(address(this))
        uint256 _amount = balanceBefore - balanceAfter
        _updatePoolBalance(poolId, pools[poolId].poolBalance + _amount);
        // transfer the loan tokens from the lender to the contract  
    }
```

Note this implementation is vulnerable to reentrancy so use a nonreentrant guard in on this function.

# **[H-03]** - Missing Slippage checks

## Summary

Users can be frontrun and receive a worse price than expected when they initially submitted the transaction.

## Vulnerability Details


The current implementation lacks essential safeguards, such as a minimum return amount or a deadline for the trade transaction to be validated. This absence of protective measures leaves the trade susceptible to front-running. It also opens up the possibility of sandwich attacks. Consequently, these vulnerabilities could result in the loss of user funds.

```solidity
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amount,
                //@audit- wtffffffff zero slippage is really bad
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });
```


## Impact

No slippage checks can lead to users being front run and lead to loss of users funds.

## Tools Used

This issue was identified through a manual review of the code.

## Recommendations

Add some sort of protection for the user such that they receive their desired amounts. Add a minimum return amount for all swap and liquidity provisions/removals to all Router functions.

# **[H-04]** - Loss of funds to protocol and lenders due to precision loss

## Summary

When the `_calculateInterest` function is called precision loss can occur if the `interestRate`, `debt`, and `timeElapsed` are too small 

## Vulnerability Details
The equation below can incur precision loss when attempting to calculate the interest and the fees for a given loan. 

Let's say the interest on a loan is set to 1%. With a debt of 5 and a time elapsed of 3 days. The math would be as follows.

100 * 5 * 10 = 50000 / 10000 = 0 / 365 days

```
        interest = (l.interestRate * l.debt * timeElapsed) / 10000 / 365 days;
        fees = (lenderFee * interest) / 10000;
        interest -= fees;
```

Leaving us with an interest rate of zero and a protocol fee of zero as well. This would allow users to take out interest-free loans.


## Impact

A user would be able to take out interest-free loans and the protocol would not collect fees either causing a loss of funds to the lenders and the protocols.

## Tools Used

Manual review.

## Recommendations

It's important to remember that the protocol's objective isn't to facilitate zero-interest loans. Therefore, verifying that neither the interest rate nor the fees are zero can help prevent precision loss and trigger a reversal if necessary. Additionally, establishing a minimum loan size could ensure that the LHS of the equation consistently exceeds the RHS.

```
 interest = (l.interestRate * l.debt * timeElapsed) / 10000 / 365 days;

        fees = (lenderFee * interest) / 10000;
        if(interest == 0 || fees == 0) revert PrecisionLoss();
        interest -= fees;

```

# **[M-01]** - Rebasing tokens will get stuck in the contract

## Summary

Rebasing tokens are tokens that have each holder's balanceof() increase over time. Aave aTokens are an example of such tokens.


## Vulnerability Details

Users expect that when they deposit tokens to a pool, they get back all rewards earned, not just a flat rate. With the contracts of this project, deposited tokens will grow in value, but the user will only be return the pre-calculated amount set in the storage variable `balances[msg.sender] -= _amount;`. Amounts go solely to the owner/creator or will remain locked in the contract if no withdraw excess tokens function is added to the contract for the owner.


```
    /// @notice withdraw tokens from stake
    /// @param _amount the amount to withdraw
    function deposit(uint _amount) external {
        TKN.transferFrom(msg.sender, address(this), _amount);
        updateFor(msg.sender);
        balances[msg.sender] += _amount;
    }
```
## Impact

If rebasing tokens are used as the collateral token, rewards accrue to the contract and cannot be withdrawn by either the user or the owner, and remain locked forever.

## Tools Used

Manual Review

## Recommended Steps

Provide a function for the pool owner to withdraw excess deposited tokens and repay any associated taxes.



# **[M-02]** - Precision loss allows users to call `giveLoans` to pools with less collateral then required

## Summary

The `_calculateInterest` function is vulnerable to precision loss. Due to dividing twice the function will return zero for both interest and `protocolInterest`. This could lead to a lender giving their loan to another pool that doesn't have the balance to cover it. Leading to a loss for them and a gain for the future pools as they will have debts greater than their balance.

## Vulnerability Details

`giveLoan()` calls `_calculateInterest()` which then returns the amount of interest the protocol fee and adds that to the loan (debt) in order to calculate the `totalDebt`.However, we already determined that these values are vulnerable to precision loss leading them to return 0 in some cases or be smaller than expected. That would lead to an inaccurate `totalDebt` and `loanRatio` allowing us to give a loan to a pool that has a higher loan ratio than our current pool which is not the expected behavior of the protocol.

## Impact

The effect of this is the `loanRatio` value will be lower than expected.

```solidity
            // assume the following
            // totalDebt  = 40 and loan.collateral = 10
            // expected calculation  30 / 10 = 3
            // actual due to precision loss 20 / 10 = 2
            // maxLoanRatio = 2
            uint256 loanRatio = (totalDebt * 10 ** 18) / loan.collateral;
            // we would expect the next line to revert but
            // this would pass allowing us to create a loan with less collateral than needed for this pool
            if (loanRatio > pool.maxLoanRatio) revert RatioTooHigh();
```

This would allow a user to move their loan to a pool in which their loanRatio is greater than the maxloanRatio for that pool. Allowing them to move and create loans to pools with less collateral than the pool requires.

## Recommended Steps

Verifying that neither the interest rate nor the fees are zero can help prevent precision loss and trigger a reversal if necessary. Additionally, establishing a minimum loan size could ensure that the left side of the equation consistently exceeds the right side.

```
 interest = (l.interestRate * l.debt * timeElapsed) / 10000 / 365 days;

        fees = (lenderFee * interest) / 10000;
        if(interest == 0 || fees == 0) revert PrecisionLoss();
        interest -= fees;

```

Another solution is creating a formula that always rounds up. This is in favor of the lender and the protocol as well. Something like this would suffice.

```
    /**
     * @param a numerator
     * @param b denominator
     * @dev Division, rounded up
     */
    function roundUpDiv(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) return 0;
        return (a - 1) / b + 1;
    }
```

# **[M-03]** - Rebasing tokens go to the pool owner, or remain locked in the various contracts

## Summary

Rebasing tokens are tokens that have each holder's balanceof() increase over time. Aave aTokens are an example of such tokens.

## Vulnerability Details

When depositing in `Lender.sol`, users expect that when they deposit tokens to a pool, they get back all rewards earned, not just a flat rate. With the contracts of this project, deposited tokens will grow in value, but the value in excess of the pre-calculated `_updatePoolBalance(poolId, pools[poolId].poolBalance + amount);` amounts go solely to the owner/creator, or will remain locked in the contract

## Impact

If rebasing tokens are used as the collateral token, rewards accrue to the contract and cannot be withdrawn by either the user or the owner, and remain locked forever.

```
    /// @notice add to the pool balance
    /// can only be called by the pool lender
    /// @param poolId the id of the pool to add to
    /// @param amount the amount to add
    function addToPool(bytes32 poolId, uint256 amount) external {
        if (pools[poolId].lender != msg.sender) revert Unauthorized();
        if (amount == 0) revert PoolConfig();
        _updatePoolBalance(poolId, pools[poolId].poolBalance + amount);
        // transfer the loan tokens from the lender to the contract
        IERC20(pools[poolId].loanToken).transferFrom(
            msg.sender,
            address(this),
            amount
        );
    }
```

## Tools Used

Manual Review

## Recommended Steps

Provide a function for the pool owner to withdraw excess deposited tokens and repay any associated taxes.


# **[L-01]** - No events are emitted in Staking.sol

## Summary

There are multiple functions that change the state that should be emitting events that in `Staking.sol`.

## Vulnerability Details

`withdraw()`
`deposit()`
`claim()`

All these and potentially others depending on the devs decision should emit events.

## Impact

The user experience is impacted by this as well as making it harder on the front end to display certain data when a function is successful.

## Tools Used

Manual review.

## Recommendations

Emit events when a state change happens.