# Audit Analysis

## Mechanism Review

**Staking Mechanism:**

- **Community Stakers:** Earn a base reward proportional to their stake. They also pay a delegation fee to Node Operator Stakers from their rewards.
- **Node Operator Stakers:** Earn both a base reward and a delegation reward proportional to their stake. The delegation reward comes from Community Stakers.

**Migration Mechanism:**

- Existing stakers from v0.1 can either migrate manually to v0.2 or withdraw their staked LINK and rewards.
- Priority access is given to v0.1 stakers for v0.2.

**Reward Mechanism:**

- Rewards in v0.2 are fixed and based on each staker's contribution.
- The reward amount remains consistent, unlike v0.1 where it was dynamic.
- Rewards can be claimed anytime but are subject to a ramp-up period.
- Early withdrawal results in a penalty, with forfeited rewards redistributed among the pool.

**Withdrawal Mechanism:**

- Staked LINK can be withdrawn post an unbonding period and within a claim period.
- If not withdrawn within the claim period, LINK is automatically restaked.
- Partial unstaking is possible, but it resets the ramp-up multiplier.
- Rewards can be withdrawn anytime without an unbonding period.

**Alerting Mechanism:**

- Alerting conditions remain the same as v0.1 at the launch of v0.2.
- Node operators have a priority round for raising alerts.
- Community Stakers can raise alerts post the priority round.
- Non-compliant Node Operator Stakers face penalties.

**Access Control Mechanism:**

- Node operators can stake anytime.
- Community Stakers have a phased rollout with priority for v0.1 stakers.
- General access is available until the v0.2 pool is filled.

**Architectural Overview:**

- Staking v0.2 adopts a modular, role-based architecture for easy upgrades.
- Core accounting logic remains in the staking contract, while other functionalities are in external contracts.
- Community Stakers use the CommunityStakingPool, while Node Operator Stakers use the OperatorStakingPool.
- RewardVault contract handles reward distribution and calculations.
- Multiple AlertsController contracts manage different alerting conditions.
- Future support for other reward mechanisms is feasible with new RewardVault versions.

### User Flows

**Node Operator Stakers (Operators) Staking Flow:**

1. **Initiation:** The operator decides to stake LINK tokens to participate in the network.
2. **Staking:** The operator sends LINK tokens to the **`OperatorStakingPool`** contract.
3. **Reward Accrual:**
    - The operator earns a base reward proportional to their stake.
    - Additionally, they earn a delegation reward derived from the Community Stakers' rewards.
4. **Alerting:** Operators monitor the ETH/USD Data Feed and can raise an alert during the priority round if the feed is down for more than 3 hours.

**Node Operator Stakers (Operators) Unstaking Flow:**

1. **Initiation:** The operator decides to unstake their LINK tokens.
2. **Unbonding Period:** After initiating the unstake, the operator waits for the unbonding period to pass.
3. **Claim Period:** Post the unbonding period, the operator can withdraw their staked LINK within the claim period.
    - If they don't withdraw within this period, the LINK is automatically restaked.
4. **Reward Withdrawal:** Operators can withdraw their rewards at any time without waiting for the unbonding period.

---

**Community Stakers Staking Flow:**

1. **Initiation:** A Community Staker decides to stake LINK tokens to participate in the network.
2. **Staking:** The Community Staker sends LINK tokens to the **`CommunityStakingPool`** contract.
3. **Reward Accrual:**
    - They earn a base reward proportional to their stake.
    - A portion of their rewards is used as a delegation fee, which goes to the Node Operator Stakers.
4. **Alerting:** After the priority round for operators, Community Stakers can raise an alert if the ETH/USD Data Feed is down for more than 3 hours and 20 minutes.

**Community Stakers Unstaking Flow:**

1. **Initiation:** The Community Staker decides to unstake their LINK tokens.
2. **Unbonding Period:** After initiating the unstake, the Community Staker waits for the unbonding period to pass.
3. **Claim Period:** Post the unbonding period, they can withdraw their staked LINK within the claim period.
    - If they don't withdraw within this period, the LINK is automatically restaked.
4. **Reward Withdrawal:** Community Stakers can withdraw their rewards at any time without waiting for the unbonding period.

## Architecture Recommendations

- I recommend rewriting the tests in the codebase to cover both happy paths and unhappy paths. While the current tests heavily lean towards testing unhappy paths, it's important to test expected and obvious outcomes as well.
- The core functionality of slash and reward should include more checks to ensure that only intended operators can be slashed. As this is a critical function of the system, all inputs should be sanitized and all potential errors should be thoroughly tested.
- There are core centralization risks throughout the protocol as the DEFAULT_ADMIN is given a lot of power to be able to call core functions in the protocol.

## The Code Analysis

### The Migration Module

***MigrationProxy.sol***

- The check for `InvalidZeroAddress()` is currently being performed four times, but the four different `if` statements can be combined into a single statement, which not only improves readability but also reduces redundancy.
- In the context of Full or Partial Migration of stakers, `OnTokenTransfer` is invoked. However, there is a missing check to ensure that `amountToStake` is not set to zero during the partial migration process. Consequently, `_migrateToPool` is called with a zero amount to stake, leading to the transfer of all LINK tokens to the users. There is no security risk as such just a missing sanity check in place.

***Migratable.sol***

- Rather than storing the old migration target in the variable and subsequently assigning the new migration target, it is recommended to emit the event first and then update the variable. Please refer to the following code example for clarification.
    
    

### **The TimeLock Module**

***StakingTimelock.sol***

- `IAccessControl.sol` is imported but never used.
- The check for `InvalidZeroAddress()` is currently being performed four times, but the four different `if` statements can be combined into a single statement, which not only improves readability but also reduces redundancy.

***Timelock.sol***

- `getMinDelay` for `s_delays` mapping will output an incorrect value if the `s_minDelay` ****is updated by the admin to a higher value, then the current `s_delays` for a function selector.
- There is a missing sanity check in the **`scheduleBatch`** function, which can only be called by the `PROPOSER_ROLE` for the `Call` array. The function assumes that the `Call` array will always have more than one item.
- There is no check in place to enforce a minimum value for `s_minDelay`. This means that the `ADMIN_ROLE` can set it to zero as well.
- There is a missing check for contract existence when making low-level calls via `.call`

### The Pools Module

***StakingPoolBase.sol***

- The two if statements that check for `minPrincipalPerStaker` can be combined into a single statement to improve readability.
- The `if` statement that reverts to `InvalidUnbondingPeriodRange` allows `MIN_UNBONDING_PERIOD == params.maxUnbondingPeriod`. Update the if greater(>) check to greaterThan(≥)
- The function `_handleOpen` defined as part of the `open` function does nothing. Its better to remove redundant code that is not required.

***CommunityStakingPool.sol***

- The removed operators are not allowed to be part of Community Staking pool based on the if condition in `_validateOnTokenTransfer`
- Rather than storing the old Operator Staking Pool target in the variable and subsequently assigning the new target, it is recommended to emit the event first and then update the variable in `setOperatorStakingPool`.

***OperatorStakingPool.sol***

- To improve code readability, errors and events should be placed in a separate file, keeping the code cleaner.
- `getRemainingSlashCapacity()` may result in potential precision loss in `slashAndReward()` if `block.timestamp` is too small. Therefore, it is important to ensure that this number is always greater than zero.
- A potential improvement for the `addOperators` function could be to perform a check to ensure that the `operators` array is not empty before proceeding with the for loop. This would help to prevent unnecessary gas usage in the case where an empty array is passed as input.
- Add a check for the `slashAndReward` `stakers` array’s input for `address(0)`, `isOperator()` checks. Ignoring such checks result in missing input validation.
- Along with `isOperator()` check for `isRemoved()`, if you don’t want the removed operators to interact with a function.
- The function used to `removeOperators()` at L474, does not removes the address of the operator from the mapping that stores the address in the `PriceFeedAlertController.sol` file which is: `s_feedSlashableOperators` . This goes against the natspec provided for the `slashAndReward` function at L271-L273.

### The Rewards Module

***RewardVault.sol***

- The check for `InvalidZeroAddress()` is currently being performed three times, but the three different `if` statements can be combined into a single statement, which not only improves readability but also reduces redundancy.
- The revert message `InvalidDelegationRateDenominator` in `setDelegationRateDenominator` can be improved to say `SameAsCurrentDelegationRateDenominator`
- The functions where in a state variable can follow the design where in the event with both old value and new value can be emitted and then move on to assign the new value instead of storing the current value in a variable. Example: `setMultiplierDuration` or `setMigrationSource`. It saves up on gas.
- The `migrate` function appears to handle the migration of rewards from one system or contract to another. It stops the vesting of rewards, encodes relevant data, and then transfers the unvested rewards to a migration target. This function has a centralization risk if `DEFAULT_ADMIN` becomes compromised.
- The `claimReward()` function claims that the caller must be a staker with a non-zero reward however it doesn’t check if the staker’s reward is greater than zero or not.
- In `finalizeReward()` the comments mention certain assumptions, like the unlikely scenario of an operator being removed after the reward vault is closed. Relying on assumptions without explicit checks can be risky.

### Learning from the Codebase

- The documentation is complete with proper natspec defined foe each of the variables and functions
- Almost 100% code coverage
- Using Typed imports throughout the codebase