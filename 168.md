SolidityATL

medium

# Vulnerable unpause logic flow lead to unfair forced liquidations

## Summary

Before pausing a protocol a position can be in good health.Once the protocol is paused that position cannot be updated. After resuming a pause, the position can immediately be liquidated if the positions falls below maintenance after settlement. This doesn't give the borrower a fair advantage to keep their collateral above 
maintenance.

## Vulnerability Detail

1. Borrower opens a position
2. Perennial admins pause the protocol for unknown reason (handle attacks, upgrades, etc)
3. Borrower is incapable of updating their position to maintain good health (good maintenance)
4. Perennial admins unpause the protocol
5. Borrower instantly becomes liquidated by adversaries or bots when protocol resumes

## Impact

Borrowers are liquidated without given the chance to credit their account to keep a healthy maintenance level on their open positions

## Code Snippet

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/collateral/Collateral.sol#L108-L135

## Tool used

Manual Review

## Recommendation

The recommendation is to incorporate a threshold period (ie. 3hrs) so borrowers can update their positions to a healthy level after the protocol is unpaused.