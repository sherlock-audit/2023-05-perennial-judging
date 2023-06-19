Brenzee

medium

# `BalancedVault.sol` does not comply with ERC4626 standard

## Summary
`BalancedVault.sol` NetSpec comments show, that BalancedVault is `ERC4626 vault that manages a 50-50 position between long-short markets of the same payoff on Perennial.`, but `BalancedVault` does not comply with ERC4626 standard.

## Vulnerability Detail
All of the missing details, what is required for an ERC4626 vault
- Doesn't have `previewDeposit`
- `deposit` has to return shares as `uint256`
- Doesn't have a `maxMint`, `previewMint`, `mint` functions
- Doesn't have `maxWithdraw`, `previewWithdraw`, `withdraw` functions
- Doesn't have `previewRedeem`, `redeem` functions

## Impact
Users who expect `BalancedVault.sol` to comply with ERC4626 standard will have limited functionality because some functions do not return correct values and some are missing.

## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/0f73469508a4cd3d90b382eac2112f012a5a9852/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L1

## Tool used
Manual Review

## Recommendation
Make sure that `BalancedVault.sol` complies with ERC4626 by fixing the listed issues.
