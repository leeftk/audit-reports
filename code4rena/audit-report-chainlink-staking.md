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
## Impact
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

## Recommended Mitigation Steps

Change `s_rewardVault.updateReward(operators[i], operatorPrincipal)` to `s_rewardVault.updateReward(operators[i], updatedPrincipal)`.
