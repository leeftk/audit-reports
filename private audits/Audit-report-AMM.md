## Overview

- Overall your contracts are extremely simple and easy to follow and there don't seem to be any major issues. The code is very well written so congrats on that!

## **[M-01]** - `swap()` function can be reentered if ERC777 tokens are used

```
    function swap(address to, bool isETHtoSPC) public payable returns (uint256) {
        uint256 sendOut;

        if (isETHtoSPC) {
            sendOut = _swap(msg.value, spcPool, ethPool + msg.value);
            spaceCoin.transfer(to, sendOut);
            emit PoolETHSwapSPC(to, msg.value, sendOut);
        } else {
            uint256 spcValue = spaceCoin.balanceOf(address(this)) - spcPool;
            sendOut = _swap(spcValue, ethPool, spcPool + spcValue);
            (bool success, ) = payable(to).call{value: sendOut}(""); 
            if (!success) revert SwapFailedETH();
            emit PoolSPCSwapETH(to, spcValue, sendOut);
        }

        ethPool = address(this).balance;
        spcPool = spaceCoin.balanceOf(address(this));

        return sendOut;
    }
```

### Summary
ERC777 allows for a callback after funds are transferred. This allows malicious attackers to reenter functions. This function does not follow CEI pattern nor does it have a nonreentrant modifier. If other ERC20 tokens are used in the future, which is the intention of this Though this is explicity this reentrancy may be able to be exploited by an attack especially considering the function is public. Try adding a reentrancy guard or implementing the check and effects interaction pattern.

## **[M-01]** - Withdraw functio does not resync balances after LP tokens are burnt

`withdraw()` should resync the reserves after an amount is sent about to the user or else the pool reserves will be inflated allowing an attacker to withdraw more than they are allowed to.

## Impact

A malicious user can completely drain the balance of the pool as when they withdraw as the reserves in the pool aren't being updated leaving the balance of the pool higher then the amount of tokens that are actually being held in the contract. This will cause any user that withdraws to get more tokens that than they should.

## Remediation 

Call the `resync()` function after each withdrawal.
