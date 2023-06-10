mstpr-brainbot

high

# Potential Vault Misaccounting due to Disproportionate Taker/Maker Ratio

## Summary
The current mechanism of equal collateral distribution between short and long products in a vault can lead to accounting discrepancies. These inconsistencies arise due to the taker/maker ratio and how funds are socialized among makers. Specifically, when the market favors long positions, the vault can incur losses due to a higher proportion of maker positions in the short market, thus reducing the vault's profit share.
## Vulnerability Detail
Currently, the vault splits collateral equally (50-50) between the short and long products of a market. Assuming the existence of only one market with two products, let's also consider an example where the price of the asset is $1 and the vault maintains a 1x leverage.

Suppose the vault holds a total of 200 units of collateral, evenly distributed between short and long markets. The vault also holds 100 units each of short and long positions in both markets.

For the sake of this example, assume the following allocations:

Short product: 100 taker units, 400 maker units (including 100 units from the vault on the maker side)
Long product: 100 taker units, 1000 maker units (including 100 units from the vault on the maker side)

Now, imagine that the asset price doubles to $2, triggering a vault and product settlement. Here's how the deltas look for both products:

Short product:

Oracle Delta: 1
Socialized Taker Delta: 100
Accumulated Position: Maker = -1/4, Taker = 1
Long product:

Oracle Delta: -1
Socialized Taker Delta: -100
Accumulated Position: Maker = 1/10, Taker = -1
Calculating the position value changes for the vault:

Short product:
The global value delta for the maker is -1/4. As the vault holds 100 positions, the collateral decrease amounts to 100 * -1/4 = -25.

Long product:
The global value delta for the maker is 1/10. As the vault also holds 100 positions here, the collateral increase amounts to 100 * 1/10 = 10.

Initially, the vault had 200 units of balance. After the calculations, the final balance is 200 - 25 + 10 = 185 units. This means that the vault lost 15 units of collateral in this oracle version for the products.

Given this scenario, vaults can continually face such accounting discrepancies due to the taker/maker ratio and the distribution of socialized funds among makers. It is evident that if the market rises (benefiting long positions), the vault will consistently incur losses because of a higher volume of maker positions in the short market, leading to a lower profit share for the vault.

## Impact
Given the scenario is very likely to happen and vault has no functionality to account the issue and also its against the vaults core idea of hedging, I'll consider it as high 
## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/types/accumulator/VersionedAccumulator.sol#L161-L175

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/types/accumulator/AccountAccumulator.sol#L28-L38

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L170-L171
## Tool used

Manual Review

## Recommendation
Also consider the taker/maker ratios on vault when rebalancing its position such that the perfect hedge exists for the vault. 