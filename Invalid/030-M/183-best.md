tvdung94

medium

# Updating collateral address in controller will cause some conflicts for the system

## Summary
Updating collateral address in controller will cause some conflicts for the system, for some contracts will not get / have new collateral address update.
## Vulnerability Detail
Controller is the contract for registering product owners, products as well as updating their settings.
However, updating collateral address cause conflict on some parts of the system
Some examples:
- Balanced vault does not update new collateral address.
- When users call withdrawFrom or any other functions required modifier settleForAccount in the old collateral contract will not get accurate results, for product.settleAccount will use new collateral address instead of the old one.
## Impact
The  system might not work as expect after updating collateral address, leading to many inconveniences.
## Code Snippet
Balanced vault only set collateral address in constructor, and there is currently no function to change collateral address.
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVaultDefinition.sol#L74-L84

modifier settleForAccount (from old collateral address) => product.settleAccount => product._settleAccount => _controller.collateral().settleAccount()  (Notice that _controller().collateral() points to the new collateral address)

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/collateral/Collateral.sol#L283-L287

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L136-L190
## Tool used

Manual Review

## Recommendation
It's quite complicated to fix this. There will be some few sub-problems when trying to fix this, for example:
- Transfering existing accounts from old collateral contract to the new one
- Transfering balance from old collateral contract to the new one
- Implementing update collateral address for balanced vault and related contracts
- etc... 
For now, the best way is not to use this function.