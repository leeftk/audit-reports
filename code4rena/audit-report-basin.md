
## **[H-01]** - Flash loan price manipulation in `Well.sol`
Line 214 of Well.sol calculates the price of tokens to tokens in the  pool based on the balances at a single point in time. 



```
    function _swapFrom(
        IERC20 fromToken,
        IERC20 toToken,
        uint256 amountIn,
        uint256 minAmountOut,
        address recipient
    ) internal returns (uint256 amountOut) {
        IERC20[] memory _tokens = tokens();
        uint256[] memory reserves = _updatePumps(_tokens.length);
        (uint256 i, uint256 j) = _getIJ(_tokens, fromToken, toToken);

        reserves[i] += amountIn;
        uint256 reserveJBefore = reserves[j];
        reserves[j] = _calcReserve(wellFunction(), reserves, j, totalSupply());

        // Note: The rounding approach of the Well function determines whether
        // slippage from imprecision goes to the Well or to the User.
        amountOut = reserveJBefore - reserves[j];
        if (amountOut < minAmountOut) {
            revert SlippageOut(amountOut, minAmountOut);
        }

        toToken.safeTransfer(recipient, amountOut);
        emit Swap(fromToken, toToken, amountIn, amountOut, recipient);
        _setReserves(_tokens, reserves);
    }
```
```
  reserves[i] += amountIn;
        uint256 reserveJBefore = reserves[j];
        reserves[j] = _calcReserve(wellFunction(), reserves, j, totalSupply());

```
Pool balances at a single point in time can be manipulated with flash loans, which can skew the numbers to the extreme. The single data point of LP balances is used to calculate the amountOut in line 231, and the amountOut influences the quantity of tokens a user receives in the transfer on line 236.

An attacker can take out a flashloan in order to increase the value of reserveJBefore. The attacker can then temporarliy increase the amout of reserveJ in the pool and then swap reserve[i] for reserveJ.`_swapFrom` will take the reservesBefore and subtract it from the current reserves. Since reserveJ is higher then usual do to the flashloan when amountOut is calculated it will cause the pool to be drained giving the attacker more tokens then then should be getting.
## Remediation 

Use a short TWAP to calculate the trade size instead of reading directly from the pool.

## **[H-02]** - Tokens with multiple addresses should be blocklisted to prevent the creation of multiple pools for the same token

Some ERC20 tokens have multiple valid contract addresses that serve as entrypoints for manipulating the same underlying storage (such as Synthetix tokens like SNX and sBTC and the TUSD stablecoin). 

```
        IERC20[] memory _tokens = tokens();
        for (uint256 i; i < _tokens.length - 1; ++i) {
            for (uint256 j = i + 1; j < _tokens.length; ++j) {
                if (_tokens[i] == _tokens[j]) {
                    revert DuplicateTokens(_tokens[i]);
                }
            }
        }
```
Its clear that the devs did not intend for the same token to be registered in the same well however this will occur if tokens with multiple addresses is used.

# References 

https://github.com/d-xo/weird-erc20#revert-on-zero-value-transfers


## Remediation

Create a blocklist for tokens that use multiple addresses and check that the tokens being set upon deployment are not on this list.


## **[M-01]** - Constant Product formula overflowing if number of tokens in reserve is too high
Line 436 in `Well.sol` calls the function `_calcLpTokenSupply()` is be called by passing the address of the well and the reserves to the `addLiquidty()` function. In the test below you can see that we set the amount of max reserves to be 100. This will cause the test to fail when trying to calculate the LP token supply.

```

        lpAmountOut = _calcLpTokenSupply(wellFunction(), reserves) - totalSupply();

```    


 However `calcLpTokenSuppl()' will revert if a big amount of tokens is passed to it. This will cause the well to be unusable and would have to be redeployed.


``` javascript


    //////////// LP TOKEN SUPPLY ////////////

    /// @dev calcLpTokenSupply: `n` equal reserves should summate with the token supply
    function testLpTokenSupplySmall(uint256 n) public {
        vm.assume(n < 100);
        vm.assume(n >= 2);
        uint256[] memory reserves = new uint256[](n);
        for (uint256 i; i < n; ++i) {
            reserves[i] = 1;
        }
        assertEq(_function.calcLpTokenSupply(reserves, _data), 1 * n);
    }

```

``` javascript

    Running 1 test for test/functions/ConstantProduct.t.sol:ConstantProductTest
    [FAIL. Reason: Arithmetic over/underflow 
    Counterexample: calldata=0x2d26b6ad000000000000000000000000000000000000000000000000000000000000004e, args=[78]] 
    testLpTokenSupplySmall(uint256) (runs: 49, μ: 21546, ~: 15383)
    Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 7.60ms
    Ran 1 test suites: 0 tests passed, 1 failed, 0 skipped (1 total tests)

    Failing tests:
    Encountered 1 failing test in test/functions/ConstantProduct.t.sol:ConstantProductTest
    [FAIL. Reason: Arithmetic over/underflow Counterexample: calldata=0x2d26b6ad000000000000000000000000000000000000000000000000000000000000004e, args=[78]] testLpTokenSupplySmall(uint256) (runs: 49, μ: 21546, ~: 15383)

    Encountered a total of 1 failing tests, 0 tests succeeded

```

As you can see from the failing test above.LP tokens out will not be able to be calculated if the amount of reserve tokens is too high causing the add liquidity function to revert after a Well contract has been deployed.

## Remediation Steps

Best way to remediate against this is to set an upper limit on the amount of tokens that can be added to a well. 

## **[M-01]** - Inflation attack in well

The `Well.sol` contract is vulnerable to a first depositor attack allowing someone to directly send funds to the pool in order to obfuscate the `totalSupply()` and steal funds from the subsequent depositor.

Below is how the attack can be carried out.

// 1. The well is empty
// 2. Alice deposits 1 token (1e18 units) into well
// 3. Bob front-runs it, depositing 1 unit
// 4. Bob donates 1 token (1e18 units) into the well using ERC20 transfer
// 5. Alice's deposit is executed

With an empty vault, the shares are being minted at a 1:1 rate with the amount. After Bob deposits 1 unit, the rate is 1:1 unit:shares.

Then, Bob donates another 1e18 units. This will make `totalAssets = 1e18 + 1`.

Finally, Alice's transaction is completed and she gets:

(1e18*1) / (1e18 + 1) = 0.99999.. shares. This will round down and cause Alice to receive 0 shares.

When Bob then withdraws he will take half of Alice's shares. Below is a POC that shows the following attack.

## Remediation 

Check if the total supply of the LP token is zero and if it is mint 1000 WEI worth of LP tokens and send it to the 0 address.

```
    /// THE FIX
    if (totalSupply == 0) {
        totalSupply = 1000;
        transfer(address(0),1000);
    }
```


## **[M-09]** - `_addLiquidity()` can fail on zero amount transfers if treasury fee is set to zero

amountIn in `_addLiquidity()` can be zero, and there is no check in place to verify that amountIn is greater than zero. Some ERC20 tokens do not allow zero value transfers, reverting such attempts.

This way, a combination of a well with this token set as one of its reserves  will revert any calls made to `_addLiquidity()` rendering contract functionality unavailable.

## References

https://github.com/d-xo/weird-erc20#revert-on-zero-value-transfers

# Remediation

Verify that amount in is greater than zero before calling `safeTransfer()`.

