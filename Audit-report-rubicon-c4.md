   # **[M-01]** - First depositor risk in reward calculation
   # Lines of code

https://github.com/code-423n4/2023-04-rubicon/blob/511636d889742296a54392875a35e4c0c4727bb7/contracts/periphery/BathBuddy.sol#L121


# Vulnerability details

## Impact
Classic issue with vaults. First Depositor can deposit a single wei then donate to the vault to greatly inflate share ratio. Due to truncation when converting to shares this can be used to steal funds from later depositors.

## Code Snippet
```   
function rewardPerToken(address token) public view returns (uint256) {
       require(friendshipStarted, "I have not started a bathToken friendship");

       if (IERC20(myBathTokenBuddy).totalSupply() == 0) {
           return rewardsPerTokensStored[token];
       }
       return
           rewardsPerTokensStored[token].add(
               lastTimeRewardApplicable(token)
                   .sub(lastUpdateTime[token])
                   .mul(rewardRates[token])
                   .mul(1e18)
                   .div(IERC20(myBathTokenBuddy).totalSupply())
           );
   }
```

## Tools Used

## Recommended Mitigation Steps
Recommended Mitigation Steps
Uniswap V2 solved this problem by sending the first 1000 LP tokens to the zero address.

```   
if (totalSupply == 0) {
       totalSupply = 1000;
};

```
# **[M-02]** - Use `safeApprove()` instead of `approve()`

# Lines of code

https://github.com/code-423n4/2023-04-rubicon/blob/511636d889742296a54392875a35e4c0c4727bb7/contracts/V2Migrator.sol#L53


# Vulnerability details

## Impact
Since SafeERC20 was imported you should use `safeApprove()` instead of `approve()`. Note that approve() will fail for certain token implementations that do not return a boolean value (). Leaving certain users unable to migrate their tokens.

## Proof of Concept

reason for setting safeApprove(0): https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit


## Tools Used

## Recommended Mitigation Steps

Use `safeApprove()` instead of `approve()`.

# **[M-03]** -`call` should be used instead of `transfer`

# Lines of code

https://github.com/code-423n4/2023-04-rubicon/blob/511636d889742296a54392875a35e4c0c4727bb7/contracts/RubiconMarket.sol#L380

https://github.com/code-423n4/2023-04-rubicon/blob/511636d889742296a54392875a35e4c0c4727bb7/contracts/RubiconMarket.sol#L381

https://github.com/code-423n4/2023-04-rubicon/blob/511636d889742296a54392875a35e4c0c4727bb7/contracts/RubiconMarket.sol#L460

https://github.com/code-423n4/2023-04-rubicon/blob/511636d889742296a54392875a35e4c0c4727bb7/contracts/RubiconMarket.sol#L461


# Vulnerability details

## Impact
The use of the deprecated transfer() function for an address will inevitably make the transaction fail when:

- The claimer smart contract does not implement a payable function.
- The claimer smart contract does implement a payable fallback which uses more than 2300 gas unit.
- The claimer smart contract implements a payable fallback function that needs less than 2300 gas units but is called through proxy, raising the call's gas usage above 2300.


Additionally, using higher than 2300 gas might be mandatory for some multisig wallets.


## Recommended Mitigation Steps

I recommend using the low-level `call()` function instead.



## **[M-05]** - `buy()` and `cancel()` ERC20 return values not checked

# Lines of code

https://github.com/code-423n4/2023-04-rubicon/blob/511636d889742296a54392875a35e4c0c4727bb7/contracts/RubiconMarket.sol#L340


# Vulnerability details


The transferFrom() function returns a boolean value indicating success. This parameter needs to be checked to see if the transfer has been successful.

Some tokens like EURS and BAT will not revert if the transfer failed but return false instead. Tokens that don’t actually perform the transfer and return false are still counted as a correct transfer.
[Here](https://gist.githubusercontent.com/lukas-berlin/f587086f139df93d22987049f3d8ebd2/raw/1f937dc8eb1d6018da59881cbc633e01c0286fb0/Tokens%20missing%20return%20values%20in%20transfer) is a list of some tokens that don't return a bool in the transfer function, though it may be a bit outdated.

## Impact
Users would be able to mint NFTs for free regardless of mint fee if tokens that don’t revert on failed transfers were used.

## Proof of Concept


## Tools Used

## Recommended Mitigation Steps
Check the success boolean of all transferFrom() calls. Alternatively, use OZ’s SafeERC20’s safeTransferFrom() function.