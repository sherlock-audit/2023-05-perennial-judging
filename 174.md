tvdung94

medium

# Users can be forced to claim assets at bad rate in some cases

## Summary
In balanced vault contract, anyone can call claim assets for other accounts; malicious users could abuse this function to force other users to receive less assets than they expect.
## Vulnerability Detail
When claiming asset, pro rate, in which users will receive less assets than expect, will be applied when total collateral is less than total unclaimed amount.  So after redeeming shares and converting it to assets, users might not want to claim assets right away in this scenario, for they will receive less token amount. However, other users can force them to claim via claim() function because there is no restrict on this function to claim for other accounts.
## Impact
Users will be forced to receive less assets in some scenarios.
## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L211-L228
## Tool used

Manual Review

## Recommendation
Only account owner (msg.sender == account) can claim asset for their account.