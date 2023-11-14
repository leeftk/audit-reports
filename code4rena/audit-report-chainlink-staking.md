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