## **[H-01]** - Malicious attacker can steal users funds if fee on transfer token is used as collateral


Some ERC20 tokens, such as USDT, allow for charging a fee any time transfer() or transferFrom() is called. If a contract does not allow for amounts to change after transfers, subsequent transfer operations based on the original amount will revert() due to the contract having an insufficient balance. However, a malicious user could also take advantage of this to steal funds from the pool.

Line [148](https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L135) in DSCEngine.sol.

```
    function depositCollateral(address tokenCollateralAddress, uint256 amountCollateral)
     //@audit- fee on transfer tokens may not work here
        public
        moreThanZero(amountCollateral)
        isAllowedToken(tokenCollateralAddress)
        nonReentrant
    {
        //This line here adds the amount a user passed as `amountCollateral` to a users totaldeposits
        s_collateralDeposited[msg.sender][tokenCollateralAddress] += amountCollateral;
        emit CollateralDeposited(msg.sender, tokenCollateralAddress, amountCollateral);
        bool success = IERC20(tokenCollateralAddress).transferFrom(msg.sender, address(this), amountCollateral);
        if (!success) {
            revert DSCEngine__TransferFailed();
        }
    }
```

```
    function _redeemCollateral(address from, address to, address tokenCollateralAddress, uint256 amountCollateral)
        private
    { 
        s_collateralDeposited[from][tokenCollateralAddress] -= amountCollateral;
        emit CollateralRedeemed(from, to, tokenCollateralAddress, amountCollateral);
        bool success = IERC20(tokenCollateralAddress).transfer(to, amountCollateral);
        if (!success) {
            revert DSCEngine__TransferFailed();
        }
    }

```

Step to Reproduce

1. Alice sends 100 of FEE token to the contract when calling `depositCollateral()`.
2. FEE token contract takes 10% of the tokens (10 FEE).
3. 90 FEE tokens actually get deposit in contract.
4. `s_collateralDeposited[msg.sender][tokenCollateralAddress] += amountCollateral;` will equal 100.
5. Attacker then sends 100 FEE tokens to the contract
6. The contract now has 180 FEE tokens but each user has an accounting of 100 FEE.
6. The attacker then tries to redeem his collateral for the full amount 100 FEE tokens.
7. The contract will transfer 100 FEE tokens to Bob taking 10 of Alices tokens with him.
8. Bob can then deposit back into the pool and repeat this until he drains all of Alice's funds.
9. When Alice attempts to withdraw the transaction will revert due to insufficent funds.

Remediation Steps

Measure the contract balance before and after the call to transfer()/transferFrom() and use the difference between the two as the amount, rather than the amount stated

```
    function depositCollateral(address tokenCollateralAddress, uint256 amountCollateral)
     //@audit- fee on transfer tokens may not work here
        public
        moreThanZero(amountCollateral)
        isAllowedToken(tokenCollateralAddress)
        nonReentrant
    {
        uint256 reserveBefore = IERC20(tokenCollateralAddress).balanceOf(address(this))
        emit CollateralDeposited(msg.sender, tokenCollateralAddress, amountCollateral);
        bool success = IERC20(tokenCollateralAddress).transferFrom(msg.sender, address(this), amountCollateral);
        if (!success) {
            revert DSCEngine__TransferFailed();
        }
        uint256 reserveAfter = IERC20(tokenCollateralAddress).balanceOf(address(this))
        amountCollateral = reserveAfter - reserveBefore;
        s_collateralDeposited[msg.sender][tokenCollateralAddress] += amountCollateral;

    }
```


## **[M-01]** - Chainlink Oracle not checking for Arbitrum Sequencer

  If the Arbitrum Sequencer goes down, oracle data will not be kept up to date, and thus could become stale. However, users are able to continue to interact with the protocol directly through the L1 optimistic rollup contract. You can review Chainlink docs on L2 Sequencer Uptime Feeds for more details on this.

As a result, users may be able to use the protocol while oracle feeds are stale. This could cause many problems, but as a simple example:

A user has an account with 100 tokens, valued at 2 USD each, and no borrows
The Arbitrum sequencer goes down temporarily
While it's down, the price of the token falls to 1 USD each
The current value of the user's account is 100 ETH, so they should be able to borrow a maximum of 50 USD to keep account healthy.
Because of the stale price, the protocol lets them borrow 100 USD.
  
  
  ```     bool isRaised = chainlinkFlags.getFlag(FLAG_ARBITRUM_SEQ_OFFLINE);
        if (isRaised) {
            // If flag is raised we shouldn't perform any critical operations
            revert("Chainlink feeds are not being updated");
        }

```




## **[M-02]** -  Rebasing tokens go to the pool owner, or remain locked in the various contracts

Summary

Rebasing tokens are tokens that have each holder's balanceof() increase over time. Aave aTokens are an example of such tokens.

Users expect that when they deposit tokens to a pool, that they get back all rewards earned, not just a flat rate. With the contracts of this project, deposited tokens will grow in value, but the value in excess of the pre-calculated `s_collateralDeposited[msg.sender][tokenCollateralAddress] += amountCollateral;` amounts go solely to the owner/creator, or will remain locked in the contract

If rebasing tokens are used as the collateral token, rewards accrue to the contract and cannot be withdrawn by either the user or the owner, and remain locked forever.

Remediation Steps

Provide a function for the pool owner to withdraw excess deposited tokens and repay any associated taxes. 




## **[L-01]** - RedeemCollateral will revert if passed an amount greater than users deposited collateral

The function `redeemCollateral()` will revert due to overflow if the user passes an amount that is greater than their current deposited collateral. Though this is intended behavior it would be useful to the user if you added a check that verfied that the amount they are attempting to withdraw is greater than or equal to the current amount deposited. This will allow you to throw a useful error message if it isn't.

The effected code is on line [282](https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L282) of `DSCEngine.sol`.


Remediation Steps

Consider adding a check that reverts if the amount being passed is greater than the users deposits for that specific token.

```
    function _redeemCollateral(address from, address to, address tokenCollateralAddress, uint256 amountCollateral)
        private
    {
        if(amountCollateral > s_collateralDeposited[from][tokenCollateralAddress]) revert NotEnoughCollateralDeposited();
        s_collateralDeposited[from][tokenCollateralAddress] -= amountCollateral;
        emit CollateralRedeemed(from, to, tokenCollateralAddress, amountCollateral);
        bool success = IERC20(tokenCollateralAddress).transfer(to, amountCollateral);
        if (!success) {
            revert DSCEngine__TransferFailed();
        }
    }

```
## **[L-02]** - Constructor should contain zero address checks for the priceFeedAddress and tokenAddress

Consider adding zero address checks to avoid redeployment.

## **[L-03]** - Precision loss if collateralValueInUSD = 1 user with 1 dollar of collateral can be liquidated due to precision loss
```
    function _calculateHealthFactor(uint256 totalDscMinted, uint256 collateralValueInUsd)
        internal
        pure
        returns (uint256)
    {
        if (totalDscMinted == 0) return type(uint256).max;
        // 100 * 1e18 / 200 = 500e16               1 * 50 = 50 / 100 = 0 
        uint256 collateralAdjustedForThreshold = (collateralValueInUsd * LIQUIDATION_THRESHOLD) / LIQUIDATION_PRECISION;
        return (collateralAdjustedForThreshold * 1e18) / totalDscMinted;
    }
```

As you can see on line 334 of `DSCengine.sol` if `collateralValueinUsd` = `1` then there will be precision loss leading to users funds getting stuck. Though it is a very minimal amount that will get stuck the user will still not be able to withdraw there 1 dollar.

Remediation Steps

Consider checking if collateralValueInUsd = 1 and if it does then return 1 instead.




## **[G-01]** - Multiple mappings can be combined into a single mapping of a value to a struct

## **[G-02]** - Cache amount value outside of loop so that you don't have to look it up in each iteration.


Line [359](https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L353) of DSCEngine.sol can be optimized. By caching uint256 `amount = s_collateralDeposited[user][token];` outside of the loop.

```
        for (uint256 i = 0; i < s_collateralTokens.length; i++) {
            address token = s_collateralTokens[i];
            uint256 amount = s_collateralDeposited[user][token];
            totalCollateralValueInUsd += getUsdValue(token, amount);
        }
```
## **[G-03]** - `++i` can be unchecked since it isn't vulnerable to overflow

Line [359](https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L353) of DSCEngine.sol can be optimized. `++i` can be unchecked as there's no risk of overflow.

```
        for (uint256 i = 0; i < s_collateralTokens.length;) {
            address token = s_collateralTokens[i];
            uint256 amount = s_collateralDeposited[user][token];
            totalCollateralValueInUsd += getUsdValue(token, amount);
            unchecked{
                ++i;
            }
        }
```



