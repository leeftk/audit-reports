**[H-1]** - DOS due to unbounded loop

## Summary

The `borrow()` function allows users to create a loan, which subsequently gets added to the `loans` array. The original design intended for a loan to be removed from the array when either `seizeLoan` or `repayLoan` is called. However, the current implementation uses the `delete` keyword, which only sets `loans[loanId]` to the default value of zero, leaving the loan in the `loans` array. If a large number of loans are created, this could cause transactions to revert, making loans impossible to repay.


## Vulnerability Details

A user's collateral could be permanently locked in the contract. The `repay` transaction would revert when called if the array length is too large due to the gas needing to iterate being greater than the block gas limit. This would render loan repayment impossible, causing a DOS. The lender would also be left without a means to liquidate the owner if another lender doesn't buy the loan, as invoking `seizeLoan` would also revert and cause a DOS.


## Impact

This would stop loans from being able to be seized. Meaning a buyer would not run the risk of being liquidated. A user wouldn't have a way to repay there loan since the transaction would always revert and cause a DOS and the lender wouldn't have a method of liquidating the owner if another lender doesn't buy the loan, as calling `seizeLoan` would also revert and cause a DOS. This would leave a users collateral forever locked in a contract.

## Tools Used

This issue was identified through a manual review of the code.

## Recommendations

To address this issue, I recommend modifying the way elements are removed from the array. Instead of using `delete`, consider using the `pop()` function. This will effectively remove the element from the array, preventing the array from growing indefinitely and mitigating the potential risks outlined above.

**[H-2]** - Missing slippage checks

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

**[H-3]** - Precision loss when calculating interest can lead to zero interest loans

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
Leaving us with an interest rate of zero and a protocol fee of zero as well. This would allow users to take out interest free loans.


## Impact

A user would be able to take out interest free loans and the protocol would not collect fees either causing a loss of funds to the lenders and the protocols.


## Recommended Steps

It's important to remember that the protocol's objective isn't to facilitate zero-interest loans. Therefore, verifying that neither the interest rate nor the fees are zero can help prevent precision loss and trigger a reversal if necessary. Additionally, establishing a minimum loan size could ensure that the left side of the equation consistently exceeds the right side.

```
 interest = (l.interestRate * l.debt * timeElapsed) / 10000 / 365 days;

        fees = (lenderFee * interest) / 10000;
        if(interest == 0 || fees == 0) revert PrecisionLoss();
        interest -= fees;

```


**[H-4]** - Loss of funds to protocol and lenders due to precision loss

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

It's important to remember that the protocol's objective isn't to facilitate zero-interest loans. Therefore, verifying that neither the interest rate nor the fees are zero can help prevent precision loss and trigger a reversal if necessary. Additionally, establishing a minimum loan size could ensure that the left side of the equation consistently exceeds the right side.

```
 interest = (l.interestRate * l.debt * timeElapsed) / 10000 / 365 days;

        fees = (lenderFee * interest) / 10000;
        if(interest == 0 || fees == 0) revert PrecisionLoss();
        interest -= fees;

```


**[H-5]** - Fee on transfer tokens can cause lender tokens to get stuck in contract.


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
`addToPool()` takes in an amount that is set by the users. `removeFromPool()` does the same, however when attempting to remove a pool using fee on transfer tokens the function would revert due to insufficent balance leaving the users funds stuck in the pool. Consider the example below.

Step to Reproduce

1. Alice sends 100 of FEE token to the contract when calling `addToPool()`.
2. FEE token contract takes 10% of the tokens (10 FEE).
3. 90 FEE tokens actually get deposit in contract.
4. `_updatePoolBalance(poolId, pools[poolId].poolBalance + amount);` will equal 100.
5. Attacker then sends 100 FEE tokens to the contract
6. The contract now has 180 FEE tokens but each user has an accounting of 100 FEE.
6. The attacker then tries to redeem his collateral for the full amount 100 FEE tokens.
7. The contract will transfer 100 FEE tokens to Bob taking 10 of Alices tokens with him.
8. Bob can then deposit back into the pool and repeat this until he drains all of Alice's funds.
9. When Alice attempts to withdraw the transaction will revert due to insufficent funds.

Remediation Steps

Measure the contract balance before and after the call to transfer()/transferFrom() and use the difference between the two as the amount, rather than the amount stated

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

**[H-6]** - Fee on transfer tokens in staking contract

## Summary

The `_calculateInterest` function is vulnerable to precision loss. Due to dividing twice the function will return zero for both interest and protocolInterest. This could lead to a lender giving there loan to another pool that doesn't have the balance to cover it. Leading to a loss for them an a gain to the future pools has they will have debts greater than their balance.

## Vulnerability Details


## Recommended Steps

Consider capping the supply or removing the mint function so that the owner has a max supply they can mint or they cannot mint anymore tokens after the contract is deployed.


## Recommended Steps

Consider capping the supply or removing the mint function so that the owner has a max supply they can mint or they cannot mint anymore tokens after the contract is deployed.


**[M-7]** - Rebasing tokens go to the pool owner, or remain locked in the various contracts

## Summary

Rebasing tokens are tokens that have each holder's balanceof() increase over time. Aave aTokens are an example of such tokens.


## Vulnerability Details

Users expect that when they deposit tokens to a pool, that they get back all rewards earned, not just a flat rate. With the contracts of this project, deposited tokens will grow in value, but the value in excess of the pre-calculated `s_collateralDeposited[msg.sender][tokenCollateralAddress] += amountCollateral;` amounts go solely to the owner/creator, or will remain locked in the contract

If rebasing tokens are used as the collateral token, rewards accrue to the contract and cannot be withdrawn by either the user or the owner, and remain locked forever.

## Tools Used

Manual Review

## Recommended Steps

Provide a function for the pool owner to withdraw excess deposited tokens and repay any associated taxes.


**[L-1]** - No events are used in Staking.sol




 **[L-2]** - Staking.sol should use more input validation to improve user experience




**[L-3]** - Seize is misspelled in the comments and in the `LoanSiezed` event

```
    event LoanSiezed(
        address indexed borrower,
        address indexed lender,
        uint256 indexed loanId,
        uint256 collateral
    );

```


```
    /// @notice sieze a loan after a failed refinance auction
    /// can be called by anyone
    /// @param loanIds the ids of the loans to sieze
```

https://word.tips/spelling/seize-vs-sieze/