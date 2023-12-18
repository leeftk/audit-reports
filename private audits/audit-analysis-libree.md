Date - 11/01/2023
## Overview:

Application for creators to create subscription services in a decentralized and permissionless way. They implement SuperFluid’s Supertokens to allow for micro payment streams directly from subscribers to creators this audit consisted primarily of the subscription service though some aspects of the lending market to build were affected by the implementation of the subscription service.

Subscriptions are created through an implementation contract called the FactoringManager.sol. Each FactoringManager deploys a vault to store Supertokens for the subscription created. A creator will then receive a Flow Token NFT. In the future, a creator can create debt against this vault and sell it on a marketplace to a vendor. 

If a Creator mints debt against a vault they will receive a debtNFT with the amount of minted debt in the metadata. The Creator can then sell or borrow against this NFT as it represents a share 

## Mechanism Review

**Subscription Mechanism:**

- **Creator Subscription:** Allows a creator to create a subscription for their services and mints a FLOW Token NFT to represent this subscription.
- **Subscriber Subscription:** A subscriber can acquire the Creator's Subscription and commit to transferring their Supertoken streams to the Vault.

**Debt Mechanism:**

- Debt can be minted against a subscription by a creator
- Debt NFT can then be sold on a marketplace
- The creator can withdraw from the vault so long as the remaining amount in the vault covers pending debt

**Vault Mechanism:**

- Vaults are currently initialized every time that a new subscription is created. The vaults store tokens and are very minimal.
- The core functionality is the `withdraw` function which can only be called by the manager.

## Architecture Recommendations

Overall the codebase is very simple and the Vault management mechanism is upgradable through the `migrate` function however it may be easier to implement the vaults as proxies instead. Each proxy can store the tokens and a vault manager would serve as the implementation contract for the proxies.

The reason for this is that migrating vaults with the current design could be unsafe for users, potentially leading to a loss of funds. At present, transactions can be front-run by attackers, enabling them to drain vaults before users can migrate. A minimal-change solution would be to introduce a batch migration for the vaults. If you detect an issue across all vaults, by the time you request users to migrate, an attacker might have already exploited the vulnerability. Let’s dive a bit deeper into this architecture below.

### In the present setup:

Users initiate a new subscription by invoking **`createSubscription`**, which calls the **`VaultFactory`** to establish a new vault. This vault falls under the management of the **`FactoringManager`** contract. Should a flaw arise in the **`FactoringManager`** contract, users would be tasked with migrating to a new manager contract individually. This is not only time-intensive but also risky; a user might not manage to migrate before an attacker exploits a bug, potentially resulting in a total loss even if a patched manager is promptly deployed.

To safeguard against this, it would be necessary to introduce a batch migration function, allowing the owner of `FactoringManager` to transition all vaults in a single transaction, similar to the function mentioned earlier.

```jsx
function batchMigrate(address[] calldata _fromList, uint256[] calldata _subscriptionList) external onlyOwner {
    require(_fromList.length == _subscriptionList.length, "Mismatched inputs length");

    for (uint256 i = 0; i < _fromList.length; i++) {
        address _from = _fromList[i];
        uint256 _subscription = _subscriptionList[i];

        Subscription memory subscription = Manager(_from).getSubscription(_subscription);

        if (Manager(_from).canMigrate(_subscription)) {
            hub.changeManagerForVault(subscription.vault, address(this));
        } else {
            revert Errors.MigrationNotPosible();
        }

        Subscription memory sub = Subscription({
            user: msg.sender,
            vault: subscription.vault,
            paymentToken: subscription.paymentToken,
            tokenId: subscription.tokenId,
            metadata: subscription.metadata,
            flowRate: subscription.flowRate,
            active: true
        });

        subscriptions[subscription.tokenId] = sub;

        Manager(_from).updateStatus(_subscription, false);

        emit SubscriptionMigrated(msg.sender, _from, address(this));
    }
}
```

This would allow the owner to migrate all the vaults safely in one transaction. If Openzeppelin’s `pausable` were used you could first pause withdrawals on all the vaults to ensure that an attacker cannot front run the batch migrate transaction and steal users’ funds. 

**Pros of Using Pausable Vaults:**

1. **Emergency Preparedness:** Enables pausing of funds in case of vulnerabilities, hacks, or economic attacks
2. **Enhanced Trust:** Builds community and user confidence 

**Cons of Using OpenZeppelin's Pausable Vaults:**

1. **Centralization Risks:** If controlled by a single entity, the pause feature can become a centralization point.
2. **User Inconvenience:** Pausing can worry users about fund accessibility 
3. **Front-run**: Front-running `batchMigrate` is still possible by an attacker

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/e8cf8601-c1b8-4e85-ad4a-e138964a2528/f07001e6-48d7-4fe3-8811-8fb5a3d0008e/Untitled.png)

### In an Optimal setup:

In the optimal setup, each vault would serve as a proxy, making delegate calls to **`FactoringManager`** for its implementation contract. This design facilitates straightforward upgrades of the Manager without requiring user involvement. Additionally, integrating **`pausable`** into this architecture would empower the admin to pause **`withdrawal`** actions during the rollout of a new implementation. This approach addresses two existing challenges: eliminating the need for users to handle migrations while ensuring vaults remain secure via **`pausable`**, and paving the way for smooth updates and feature additions.

Below is a diagram illustrating the concept of employing proxies for the vaults. Compared to prior designs, this configuration promotes effortless upgrades whenever protocol modifications or bug fixes are necessary.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/e8cf8601-c1b8-4e85-ad4a-e138964a2528/6007e247-25ea-4b1d-bd76-95870398dcb6/Untitled.png)

**Pros of Using Proxies:**

1. **Emergency Preparedness:** Enables swift response to vulnerabilities, hacks, or economic attacks.
2. **Operational Flexibility:** Assists in managing upgrades.
3. **Enhanced Trust:** Builds community and user confidence.
4. **User Convenience:** Proxies will lead to a minimal amount of downtime since users will not need to upgrade themselves.
5. Front-run protection: Front-running is not possible since only the owner can update the implementation contract.

**Cons of Using OpenZeppelin's Proxies:**

1. **Centralization Risks:** If controlled by a single entity, the pause feature can become a centralization point as can the proxy itself
2. **Proxy Risks:** Proxies come with their own set of risks, if not implemented properly they can introduce new bugs.

**Other areas of concern:**

- TokenIDs begin at 1 during minting, but elsewhere in the code, they are considered to start at 0. While not problematic now, this inconsistency might cause issues if future features rely on the tokenIds.
- In NFTToken.sol, there's an implemented **`_burn`** function, but it's unused. Instead, the **`burn()`** function from the inherited ERC721 contract is utilized.
- It's been proposed to use Pausable contracts for the vaults, enabling the owner to halt the vaults. This is the most secure approach under the current circumstances.