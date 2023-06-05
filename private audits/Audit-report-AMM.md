### Overview

- Overall your contracts are extremely simple and easy to follow and there doesn't seem to be any major issues. The code is very well written so congrats on that!

**[M-01]** - Potential reentrancy in withdraw function SpaceLP line 86.

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

In this function you seem to be updating the state of the balances after you perform the swap. Though this is explicity this reentrancy may be able to be exploited by an attacked especially considering the function is public. Try adding a reentrancy guard or implementing the check and effects interaction pattern.

**[M-01]** - Withdraw function on line 63 does not resynce balances after Lptokens are burnt

`withdraw()` should resync the reserves after an amount is sent about to the user or else the pool reserves will be inflated allowing an attacker to withdraw more than they are allowed to.