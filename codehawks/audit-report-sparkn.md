# Sparkn  - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Lack of chainID validation allows signatures to be re-used across forks](#M-01)
    - ### [M-02. Immutable `STADIUM_ADDRESS` Puts Funds at Risk](#M-02)
- ## Low Risk Findings
    - ### [L-01. Events are not `indexed`](#L-01)
    - ### [L-02. Spelling error in the comments](#L-02)
    - ### [L-03. Use 2 step Ownable by OpenZeppelin](#L-03)
    - ### [L-04. Precision loss/Rounding to Zero in `_distribute()`](#L-04)
    - ### [L-05. `Distributor.sol` will DOS when winners are abnormally high due to no restriction on winners' array](#L-05)
    - ### [L-06. Funds can get stuck in contract if a winner is on the USDT blocklist](#L-06)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: CodeFox Inc.

### Dates: Aug 21st, 2023 - Aug 29th, 2023

[See more contest details here](https://www.codehawks.com/contests/cllcnja1h0001lc08z7w0orxx)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 2
   - Low: 6



		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Lack of chainID validation allows signatures to be re-used across forks            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L152

## Summary

If the chain forks after deployment, the signature passed when calling `deployProxyAndDistributeBySignature()` may be considered valid on both forks.


## Vulnerability Details
The signatures used to deployProxy and distribute rewards do not account for chain splits. The chainID is not included in the domain separator. As a result, if the chain forks after deployment, the signed message may be considered valid on both forks.

Imagine Bob as one of the winners. An EIP is included in an upcoming hard fork that has split the community. After the hard fork, a significant user base remains on the old chain. On the new chain, The organizer calls `deployProxyAndDistributeBySignature`. Bob(the malicious user), operating on both chains, replays the signature on the old chain and is able to deploy the proxy and steal funds

## Impact
The protocol becomes vulnerable to Signature Replay attacks where in malicious users replay the signatures to steal funds from the organizers.

## Tools Used

Manual Review

## Remediation Steps
Include the chainID in the signature schema. This will make replay attacks impossible in the event of a post-deployment hard fork
## <a id='M-02'></a>M-02. Immutable `STADIUM_ADDRESS` Puts Funds at Risk            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/Distributor.sol#L58C2-L58C2

## Summary

The `STADIUM_ADDRESS` is hardcoded in the implementation contract.

## Vulnerability Details

The `STADIUM_ADDRESS` is supposed to receive 5% of the rewards on every proxy deployment.

Suppose `STADIUM_ADDRESS` gets compromised, and the private key is exposed or hacked, enabling the hacker to run a bot to transfer all incoming funds to another address under their control. Unfortunately, there is no mechanism for the organizer or the proxy factory owner to alter `STADIUM_ADDRESS` address. As a result, all the reward fees is now sent to the hacker controlled address.

To address this vulnerability, one potential solution is to deploy a new implementation with an alternative `STADIUM_ADDRESS` or define a function called `changeStadiumAddress` wherein the `STADIUM_ADDRESS` can be changed. 

However, it's important to note that contracts deployed with the old `STADIUM_ADDRESS` will remain susceptible.

##Impact

The funds designated for rewards fees could be diverted to unauthorised accounts.

## Tools Used

Manual Review

## Remediation Steps

Writing a function to update the `STADIUM_ADDRESS` which will help resolve this vulnerability. The caller of the function should be factory address or owner of the factory.

Example:

```
function changeStadiumAddress(address newStadiumAddress) external {
    require(newStadiumAddress != address(0), "New address cannot be zero");
    if (msg.sender != FACTORY_ADDRESS) {
            revert Distributor__OnlyFactoryAddressIsAllowed();
    }
    STADIUM_ADDRESS = newStadiumAddress;
    emit StadiumAddressChanged(newStadiumAddress);
}
```

# Low Risk Findings

## <a id='L-01'></a>L-01. Events are not `indexed`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/Distributor.sol#L64

## Summary

The emitted events are not `indexed`, making off-chain scripts such as front-ends of dApps to filter the events efficiently.

## Vulnerability Details

Indexed event fields make the field more quickly accessible to off-chain
tools that parse events like The Graph. However, note that each index field costs extra gas during emission, so it’s not necessarily best to index the maximum
allowed per event (three fields)


## Tools Used

Manual Review

## Remediation Steps

Add the indexed keyword in each event, e.g., event Distributed(address token, address[] winners, uint256[] percentages, bytes data);.
## <a id='L-02'></a>L-02. Spelling error in the comments            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/Distributor.sol#L114

## Summary

Misspelled words in comments.

`* @param data The data to be logged. It is supposed to be used for 
showing the realation bbetween winners and proposals.`

## Vulnerability Details

The parameter description for the variable data contains a typo, where 
"realation bbetween" should be corrected to "relation between". 

## Tools Used

Manual Review
## <a id='L-03'></a>L-03. Use 2 step Ownable by OpenZeppelin            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L26

## Summary
The contracts `ProxyFactory.sol` does not implement a 2-Step-Process for transferring ownership which can result in lose of ownership.

## Vulnerability Details

Ownership of the contract can easily be lost when making a mistake when transferring ownership. Since the privileged roles have critical function roles assigned to them. Assigning the ownership to a wrong user can be disastrous.

So Consider using the `Ownable2Step` contract from OZ (https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol) instead. The way it works is there is a `transferOwnership` to transfer the ownership and `acceptOwnership` to accept the ownership.




##Impact

If ownership is lost, The `onlyOwner` functions like `setContest`, `deployProxyAndDistributeByOwner` and `distributeByOwner` will be inaccessible.

## Tools Used

Manual Review

## Remediation Steps

Implement 2-Step-Process for transferring ownership via Ownable2Step contract from OpenZeppelin (https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol)
## <a id='L-04'></a>L-04. Precision loss/Rounding to Zero in `_distribute()`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/Distributor.sol#L144

## Summary

The identified vulnerability is associated with the `_distribute` function in 
`Distributor.sol`. In scenarios where the total token amount is low or low values are used for percentages, the function may encounter a precision issue. 
This arises due to the division of totalAmount by `BASIS_POINTS` to calculate the distribution amount for each winner. The precision error can lead to incorrect token distribution, affecting the fairness and accuracy of rewards to winners.

## Vulnerability Details

The vulnerability stems from the calculation of `amount` within the distribution loop. The formula `amount = totalAmount * percentages[i] / BASIS_POINTS` involves a division operation that could result in loss of precision(Rounding to Zero) when dealing with small `totalAmount` values or low `percentages[i]`. This imprecision can lead to token amounts being rounded down to zero, resulting in unfair or incomplete rewards for winners.

### Proof Of Concept:

To simulate the vulnerability we need to make changes in the modifier 
`setUpContestForJasonAndSentJpycv2Token`:

Code:

```solidity

	modifier setUpContestForJasonAndSentJpycv2Token(address _organizer) {
        vm.startPrank(factoryAdmin);
        bytes32 randomId = keccak256(abi.encode("Jason", "001"));
        proxyFactory.setContest(_organizer, randomId, block.timestamp + 8 days, address(distributor));
        vm.stopPrank();
        bytes32 salt = keccak256(abi.encode(_organizer, randomId, address(distributor)));
        address proxyAddress = proxyFactory.getProxyAddress(salt, address(distributor));
        vm.startPrank(sponsor);
        MockERC20(jpycv2Address).transfer(proxyAddress, 10);
        vm.stopPrank();
        // console.log(MockERC20(jpycv2Address).balanceOf(proxyAddress));
        // assertEq(MockERC20(jpycv2Address).balanceOf(proxyAddress), 10000 ether);
        _;
    }

```

We change the value transferred to the `proxyAddress` to `10` tokens.

Note: We are not using `10 ether` here as ether has 18 decimals which misguides the intended attack.

Now, we create a function called `testPrecisionLoss()` wherein we simulate end of a contest and call the `deployProxyAndDistribute()` function. This makes use of the modified `createData()` function to send in `95` winners, each being rewarded with percentage of `100 BASIS POINTS`, which is equal to `(10000 - COMMISSION_FEE)` i.e. `9500 BASIS POINTS` thus satisfying the conditions in the `_distribute()` function and allowing distribution of funds.

Code:
```
function testPrecisionLoss() public setUpContestForJasonAndSentJpycv2Token(organizer) {
        bytes32 randomId_ = keccak256(abi.encode("Jason", "001"));

        //create data from modified createData() function
				bytes memory data = createData();

        vm.startPrank(organizer);
        console.log("User1 Start Balance -", MockERC20(jpycv2Address).balanceOf(user1));
        console.log("Stadium Balance Before: ",MockERC20(jpycv2Address).balanceOf(stadiumAddress));

        //warping to the time where contest ends and token distribution is allowed
				vm.warp(30 days);

				// distributing the rewards to all 95 winners 
        proxyFactory.deployProxyAndDistribute(randomId_, address(distributor), data);

        console.log("Stadium End Balance: ",MockERC20(jpycv2Address).balanceOf(stadiumAddress));
        console.log("User1 After Balance -", MockERC20(jpycv2Address).balanceOf(user1));

        vm.stopPrank();
    }
```

The logs prove the existence of precision loss:

```
Running 1 test for test/integration/ProxyFactoryTest.t.sol:ProxyFactoryTest
[PASS] testPrecisionLoss() (gas: 892788)
Logs:
  0x000000000000000000000000000000000000000E
  User1 Start Balance - 0
  Stadium Balance Before:  0
  Stadium End Balance:  10
  User1 After Balance - 0

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.16ms
```

As we can see, all of the balance gets transferred to the `stadiumAddress` from the contract as the `_commissionTransfer()` function declares:

```
function _commissionTransfer(IERC20 token) internal {
        token.safeTransfer(STADIUM_ADDRESS, token.balanceOf(address(this)));
    }
```

Due to the precision loss(Rounding to Zero), none of the amount gets transferred to any winners and thus, all the "remaining tokens" get sent to the `stadiumAddress`, thus leaving our winners - in this case, user1 - rewarded with 0 balance.

## Impact

The Rounding to Zero vulnerability has the potential to undermine the intended fairness and accuracy of the reward distribution process. In scenarios where the token balance is very small or percentages are low, the distribution algorithm could yield incorrect or negligible rewards to winners. This impacts the trust and credibility of the protocol, potentially leading to user dissatisfaction and decreased participation.

## Tools Used

Manual Review
Foundry
VSCode

## Recommendations

Consider instituting a `predefined minimum threshold` for the `percentage` amount used in the token distribution calculation. This approach ensures that the calculated distribution amount maintains a reasonable and equitable value, even when dealing with low percentages. 

Additionally, an alternative strategy involves adopting a `rounding up equation` that consistently rounds the `distribution amount upward` to the nearest integer value. 

By incorporating either of these methodologies, the precision vulnerability associated with small values and percentages can be effectively mitigated, resulting in more accurate and reliable token distribution outcomes.
## <a id='L-05'></a>L-05. `Distributor.sol` will DOS when winners are abnormally high due to no restriction on winners' array            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/Distributor.sol#L144

## Summary

The vulnerability pertains to the `_distribute` function within the protocol. In 
scenarios where a large number of winners is permitted, the function's execution 
may iterate over an extensive array of data, causing potential performance degradation. Notably, the absence of an upper limit on the number of winners could result in a Denial of Service (DoS) scenario, impairing the core functionality of the protocol – the fair distribution of rewards to the designated winners. Consequently, this vulnerability could undermine the protocol's operational efficiency and reliability.

## Vulnerability Details

We need a set of values for winners array and percentages array, that satisfies 
the conditions in the function and makes the length of the array very long. 
COMMISSION_FEE is 500 and BASIS_POINTS is 10000.


## Impact
The following alteration of the `createData()` function allows us to make a `winners` array of 10,000 users. From there, we proceed to create a `percentages_` array of 10,000 elements as well.

Furthermore, we run a `for` loop to equally distribute the percentages among the winners array. This could also be an uneven, as long as the sum adds up to `9500 Points`.

So now every winner has a corresponding percentage of `0.95%`.

```
function createData() public view returns (bytes memory data) {
        address[] memory tokens_ = new address[](1);
        tokens_[0] = jpycv2Address;
        address[] memory winners = new address[](10000);
        winners[0] = user1;
        uint256[] memory percentages_ = new uint256[](10000);
        for (uint256 i; i < percentages_.length; i++) {
            percentages_[i] = 9500/10000;
        }
        data = abi.encodeWithSelector(Distributor.distribute.selector, jpycv2Address, winners, percentages_, "");
    }
```

To demonstrate the impact, we can run any test case in the pre-defined tests that uses the `createData()` function, to prove that the loop iteration uses large amount of gas as compared to having a small number of winners.


## Tools Used

Manual Review

## Recommendations

* Implement a Cap on Winners: Introduce an upper limit on the number of winners for any given contest in the `Distributor.sol` contract that can be included in a distribution. This cap should be determined based on the protocol's capacity and resources, aiming to strike a balance between fair distribution and system performance.


## <a id='L-06'></a>L-06. Funds can get stuck in contract if a winner is on the USDT blocklist            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/Distributor.sol#L116

## Summary
The provided code snippet contains a vulnerability that could potentially lead to trapped funds in a contract. The vulnerability arises in the `_distribute()` function, where the `_distrubute()` function will loop through the `winners[]` array and pay them out accordingly. For tokens such as USDT, USDC, a contract-level admin-controlled address blocklist feature exists, malicious or compromised token owners could block the contract's address. This would cause the entire transaction to revert, leaving the funds trapped within the contract.

## Vulnerability Details

The vulnerability occurs in the `_distribute()` function, specifically in the loop that transfers tokens to the winners. Since the loop iterates over an array of winners' addresses and performs token transfers to these addresses, if any of the winners' addresses are blocked by the USDT's blocklist, the transfer will be forbidden, and the transaction will revert. This would result in funds becoming stuck in the contract, and the intended distribution of tokens would fail.

## Impact
The problem with this is that if an address in the array is on the USDT block list the entire transaction will revert leading to a DOS of the contract and all of the funds stuck.

## Tools Used

Manual Review

## Recommendations

Possible solutions:
1. Implement 2-step Withdrawals:
- Users call the safeWithdraw function with their address as an argument.
- The function verifies the user's address and their non-zero balance.
- If both conditions are met, the function transfers the allocated tokens from the contract to the user's address.

2. Skip Blacklisted Users:
- Save all the blacklisted address in a variable.
- Skip those addresses in the processWithdrawals loop.

3. Address Verification: Prior to performing any token transfers in the _distribute 
function, validate each winner's address against the token's blocklist. If any address 
is found to be blocked, skip the transfer for that specific winner.


