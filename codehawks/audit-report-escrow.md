**[M-1]** - USDC blacklisted accounts can DoS the withdrawal system

When resolve dispute is called tokens are sent to the buyer, seller and arbiter. However if any of these tokens are only the USDC blacklist the transaction will fail leaving the seller with none of their tokens being transferred and tokens getting locked in the contract. Considering USDC is a commonly used ERC20 for these types of services its safe to assume that this could cause some tokens to get stuck in escrow.

Impact

Tokens can get locked in the contract.

Tools Used

Manual Review

Remediation Steps

Instead of sending tokens directly to the payer or recipient in cancel(), consider storing the number of tokens in variables and having the payer or recipient claim it later

**[L-01]** - Reentrancy guard uneccesary 

The docs clearly state that ERC777 should be discourage due to a risk of DOS attacks. The 'resolveDispute' function follows the checks effects interaction pattern and so reentrancy is currently not possible.

Consider removing the nonReentrant and non importing the reentrancy guard contracts as it is a waste of gas.

"Tokens with callbacks allow malicious sellers to DOS dispute resolutions - Each supported token will be vetted to be supported. ERC777 should be discouraged."ß


https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/Escrow.sol#L109

**[L-01]** - Inconsistent reentrancy guard

The contract has an inconsistent use of the nonReentrant modifier using it on the `resolveDispute()` function but not on the `confirmReceipt()` function. Both function call the `safeTransfer` method and so in this case they either both should have the modifier or neither should. Considering that they follow the CEI pattern, they both have modifier only allowing specific users to transfer funds and neither of them can be reentered unless using ERC777 which in this case has be stated as a known issue and is discouraged.

**[G-01]** - `buyerAward != 0` is cheaper than `if (buyerAward > 0)` 
Note: Minor optimization, the amount of gas saved is minor, change when you see fit.

It is cheaper to use != 0 than > 0 for uint256.

```solidity
        if (buyerAward > 0) {
            i_tokenContract.safeTransfer(i_buyer, buyerAward);
        }
        if (i_arbiterFee > 0) {
            i_tokenContract.safeTransfer(i_arbiter, i_arbiterFee);
        }
        tokenBalance = i_tokenContract.balanceOf(address(this));
        if (tokenBalance > 0) {
            i_tokenContract.safeTransfer(i_seller, tokenBalance);
        }
```

**[G-02]** - setting the constructor to 'payable'

You can cut out 10 opcodes in the creation-time EVM bytecode if you declare a constructor payable. Making the constructor payable eliminates the need for an initial check of msg.value == 0 and saves 13 gas on deployment with no security risks.


'''
Before optimization 
[PASS] testCanOnlyConfirmInCreatedState() (gas: 767482)
[PASS] testCanOnlyInitiateDisputeInConfirmedState() (gas: 766427)
[PASS] testCanOnlyResolveInDisputedState() (gas: 774477)
'''

'''
After optimization
[PASS] testCanOnlyConfirmInCreatedState() (gas: 767467)
[PASS] testCanOnlyInitiateDisputeInConfirmedState() (gas: 766412)
[PASS] testCanOnlyResolveInDisputedState() (gas: 774462)
'''


**[G-03]** - Using clones is cheaper than using create2

There’s a way to save a significant amount of gas on deployment using Clones: https://www.youtube.com/watch?v=3Mw-pMmJ7TA .

This is a solution that was adopted, as an example, by Porter Finance. They realized that deploying using clones was 10x cheaper:

https://github.com/porter-finance/v1-core/issues/15#issuecomment-1035639516
https://github.com/porter-finance/v1-core/pull/34
I suggest applying a similar pattern, here with a cloneDeterministic method to mimic the current create2


