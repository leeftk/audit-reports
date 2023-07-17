## **[H-01]** -  Users can `mint()` NFTs for the the lowest price possible (endPrice)  
## Summary

The constructor doesn't force the start time to be before block.timestamp meaning a person can accidentally create an auction with a block.timestamp in the past. This would cause the mint function to revert when attempting to calculate the timePassed [here](https://github.com/sherlock-audit/2023-07-beam-auction-leeftk/blob/81b3ba8fa7c1aa1795721cdca71c1f28d8186869/dutch-nft/src/MeritDutchAuction.sol#L116)

## Vulnerability Detail
```
        uint256 timePassed = time - startTime;
        uint256 totalSteps = (endTime - startTime) / stepSize;
        uint256 currentStep = timePassed / stepSize;
        uint256 priceRange = startPrice - endPrice;

        // Calculate the price at the current step
        uint256 price = startPrice - (priceRange * currentStep / totalSteps);
        return price;
```
Let's say the contract was just deployed and the startTime was set to exactly 7 days ago.

timePassed = Today - startTime(1 week ago) = 7 days
totalSteps = 7 (step size 1 day)
currentStep = 7 days / 1 day = 7 or the last step
priceRange = 100 - 1 = 99

price = 100 - (99 * 7 / 7) = 1

This would allow the user to then purchase the NFT for the lowest price possible. Essentially giving them a huge discount and causing a huge loss to the protocol.

## Impact

Let's say the contract was just deployed and the startTime was set to exactly 7 days ago.

timePassed = Today - startTime(1 week ago) = 7 days
totalSteps = 7 (step size 1 day)
currentStep = 7 days / 1 day = 7 or the last step
priceRange = 100 - 1 = 99

price = 100 - (99 * 7 / 7) = 1

This would allow the user to then purchase the NFT for the lowest price possible. Essentially giving them a huge discount and causing a huge loss to the protocol.

## Code Snippet

Below is the code the uses the getPrice() function in order to price NFTs based on the current step.

```
        uint256 pricePaid = getPrice();
        // Total price being paid is the current price times the amount of NFTs
        uint256 totalPaid = pricePaid * amount;
        // Auction must have started
        require(block.timestamp >= startTime, "Auction not started");
        // Cannot mint more than the mint cap
        require(mintedBefore + amount <= MINT_CAP, "Mint cap reached");
        // Cannot mint more than the cap per address
        require(mintedPerAddressBefore + amount <= CAP_PER_ADDRESS, "Mint cap per address reached");
        // Make sure enough ETH was sent in the transaction
        require(msg.value >= totalPaid, "Insufficient funds");

```

You can see in the mint function that the user is required to send more ether then the total amount they've spent (`totalPaid`) so far. However `totalPaid` is being calculated by taking the amount of NFTs the user wants to mint and multiplying it by the return value of `getPrice()` which in this case would be the `endPrice` since the `startTime` was set in the past before the current block.timestamp.
 
## Tool used

Manual Review

## Recommendation

In the constructor add a require statement that forces the user to pass a startTime that is after the block.timestamp.

`require(block.timestamp <= startTime)`


## **[M-01]** -  Use `safeMint()` instead of `_mint()`
## Summary

Use safeMint instead of mint for ERC721

## Vulnerability Detail

Vulnerability Detail
The msg.sender will be minted as a proof of staking NFT when _stakeToken() is called.

However, if msg.sender is a contract address that does not support ERC721, the NFT can be frozen in the contract.

As per the documentation of EIP-721:

A wallet/broker/auction application MUST implement the wallet interface if it will accept safe transfers.

Ref: https://eips.ethereum.org/EIPS/eip-721

As per the documentation of ERC721.sol by Openzeppelin

https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/ERC721.sol#L274-L285

The lines of code where the issue appears below:

Auction contract:
https://github.com/sherlock-audit/2023-07-beam-auction-leeftk/blob/81b3ba8fa7c1aa1795721cdca71c1f28d8186869/dutch-nft/src/MeritDutchAuction.sol#L158

MeritNFT:
https://github.com/sherlock-audit/2023-07-beam-auction-leeftk/blob/81b3ba8fa7c1aa1795721cdca71c1f28d8186869/dutch-nft/src/MeritNFT.sol#L62

## Impact

Users possibly lose their NFTs

## Code Snippet

```solidity
    /// @notice Mints an NFT. Can only be called by an address with the minter role and tokenId must be unique
    /// @param _tokenId Id of the token
    /// @param _receiver Address receiving the NFT
    function mint(uint256 _tokenId, address _receiver) external onlyMinter {
        _mint(_receiver, _tokenId);
    }
```

## Tool used

Manual Review

## Recommendation

Use `safeMint` instead of mint to check received address support for `ERC721` implementation.



## Summary

If startTime is set to a date far in the past `mint()` can underflow and lead to nfts being stuck in the contract and the `mint()` never fully executing.

## Vulnerability Detail

We have already determined that the block.timestamp can be set to a date in the past and this can allow a user to manipulate the currentStep. If the currentStep is too large it will cause the price variable to underflow and increase to the maximum possible value (2^256) - 1. This would cause the `_mint()` function to revert as the price will always be greater than any user can afford and so it would render this contract useless and all of the NFTs would be stuck in the MeritNFT contract since only the auction contract can call the NFT contract. 


## Impact
NFTs will be stuck as the transaction will always revert due to underflow in the`mint()` function.

## Code Snippet
The code below is a slightly modified version of the code for simplicity's sake however the reference is here. [LINK](https://github.com/sherlock-audit/2023-07-beam-auction-leeftk/blob/81b3ba8fa7c1aa1795721cdca71c1f28d8186869/dutch-nft/src/MeritDutchAuction.sol#L122C59-L122C59)

```solidity
    uint256 timePassed = time - startTime;
    uint256 totalSteps = (endTime - startTime) / stepSize;
    uint256 currentStep = timePassed / stepSize;
    uint256 priceRange = startPrice - endPrice;

    uint256 price = startPrice - (priceRange * currentStep / totalSteps);
    return price;
    
```

As you can see above timePassed is calculated by taking the block.timestamp and subtracting it from the startTime. However, since the startTime is not required to be greater than the block.timestamp when the contract is deployed we can set a startTime to a point in history in the past allowing us to manipulate and artificially increase it.

Due to Solidity 0.8 default checked math, the subtracting `(priceRange * currentStep / totalSteps)`  from `startPrice` will cause a negative value that will generate an underflow in the unsigned integer type and lead to a transaction revert.

Calls to getPrice will revert, and since this function is used in `mint()` to calculate the current NFT price it will also cause the `mint()` process to fail. The price drop will continue to increase as time passes, making it impossible to recover from this situation and effectively bricking the contract.

This will eventually lead to a loss of funds and stuck NFTS in the contract.

This is different from the other issue reported around setting dates in the past as this one would render the contract unusable if the startTime is too far in the past while the other is related to setting the startTime to the right moment in the past so that users can mint NFTs for the `endPrice` both of these assume the deployed fat fingers the startTime when deploying the contract as they wouldn't have the incentive to do this intentionally.

## Tool used

Manual Review

## Recommendation
Force the start time to be greater or equal to block.timestamp.

```solidity
require(block.timestamp <= startTime)
```



## **[M-02]** - Lack of zero address checks could cause NFTs to get stuck in contract

Normally I wouldn't consider a lack of zero address validation a medium however in the case of this contract it can actually lead to NFTs get stuck in the contract forever and render both of the contract useless.