mstpr-brainbot

high

# Vaults can't handle the incentives distributed by the products

## Summary
Vaults are eligible for product incentive rewards but lack the necessary functions to claim and distribute these rewards to users within the vault. 
## Vulnerability Detail
In the current scenario, if a product has an incentiviser, all depositors are eligible for these incentives. As vaults also make deposits into products, they are also qualified to claim rewards from the incentiviser. However, the vaults lack the functionality to claim these rewards from the incentiviser, and there isn't any existing code to distribute these rewards to the users within the vault. Even when a vault does not participate in the incentives, its presence still dilutes the share of rewards for other product users in the reward pool.
## Impact
Although the rewards can be claimed for anyone, vault has no functionality to handle the reward tokens. This will create an economical damage to reward depositor (program owner) and the users in the product hence, I'll call it as high.
## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L167
## Tool used

Manual Review

## Recommendation
Implement functionality allowing vault depositors to claim product-related rewards if they're eligible. If vaults are excluded from such rewards, their deposited balance should not factor into the incentiviser's distribution calculations, thereby increasing other product depositors' reward shares.