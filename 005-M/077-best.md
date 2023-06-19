Emmanuel

medium

# User would liquidate his account to sidestep `takerInvariant` modifier

## Summary
A single user could open a massive maker position, using the maximum leverage possible(and possibly reach the maker limit), and when a lot of takers open take positions, maker would liquidate his position, effectively bypassing the taker invariant and losing nothing apart from position fees.
This would cause takers to be charged extremely high funding fees(at the maxRate), and takers that are not actively monitoring their positions will be greatly affected. 

## Vulnerability Detail
In the closeMakeFor function, there is a modifier called `takerInvariant`.
```solidity
function closeMakeFor(
        address account,
        UFixed18 amount
    )
        public
        nonReentrant
        notPaused
        onlyAccountOrMultiInvoker(account)
        settleForAccount(account)
        takerInvariant
        closeInvariant(account)
        liquidationInvariant(account)
    {
        _closeMake(account, amount);
    }
```
This modifier prevents makers from closing their positions if it would make the global maker open positions to fall below the global taker open positions.
A malicious maker can easily sidestep this by liquidating his own account.
Liquidating an account pays the liquidator a fee from the account's collateral, and then forcefully closes all open maker and taker positions for that account.
```solidity
function closeAll(address account) external onlyCollateral notClosed settleForAccount(account) {
        AccountPosition storage accountPosition = _positions[account];
        Position memory p = accountPosition.position.next(_positions[account].pre);

        // Close all positions
        _closeMake(account, p.maker);
        _closeTake(account, p.taker);

        // Mark liquidation to lock position
        accountPosition.liquidation = true; 
    }
```
This would make the open maker positions to drop significantly below the open taker position, and greatly increase the funding fee and utilization ratio.

### ATTACK SCENARIO
- A new Product(ETH-Long) is launched on arbitrum with the following configurations:
    - 20x max leverage(5% maintenance)
    - makerFee = 0
    - takerFee = 0.015
    - liquidationFee = 20%
    - minRate = 4%
    - maxRate = 120%
    - targetRate = 12%
    - targetUtilization = 80%
    - makerLimit = 4000 Eth
    - ETH price = 1750 USD
    - Coll Token = USDC
    - max liquidity(USD) = 4000*1750 = $7,000,000
- Whale initially supplies 350k USDC of collateral(~200ETH), and opens a maker position of 3000ETH($5.25mn), at 15x leverage.
- After 2 weeks of activity, global open maker position goes up to 3429ETH($6mn), and because fundingFee is low, people are incentivized to open taker positions, so global open taker position gets to 2743ETH($4.8mn) at 80% utilization. Now, rate of fundingFee is 12%
- Now, Whale should only be able to close up to 686ETH($1.2mn) of his maker position using the `closeMakeFor` function because of the `takerInvariant` modifier.
- Whale decides to withdraw 87.5k USDC(~50ETH), bringing his total collateral to 262.5k USDC, and his leverage to 20x(which is the max leverage)
- If price of ETH temporarily goes up to 1755 USD, totalMaintenance=3000 * 1755 * 5% = $263250. Because his totalCollateral is 262500 USDC(which is less than totalMaintenance), his account becomes liquidatable.
- Whale liquidates his account, he receives liquidationFee*totalMaintenance = 20% * 263250 = 52650USDC, and his maker position of 3000ETH gets closed. Now, he can withdraw his remaining collateral(262500-52650=209850)USDC because he has no open positions.
- Global taker position is now 2743ETH($4.8mn), and global maker position is 429ETH($750k)
- Whale has succeeded in bypassing the takerInvaraiant modifier, which was to prevent him from closing his maker position if it would make global maker position less than global taker position.

Consequently,
- Funding fees would now be very high(120%), so the currently open taker positions will be greatly penalized, and takers who are not actively monitoring their position could lose a lot.
- Whale would want to gain from the high funding fees, so he would open a maker position that would still keep the global maker position less than the global taker position(e.g. collateral of 232750USDC at 15x leverage, open position = ~2000ETH($3.5mn)) so that taker positions will keep getting charged at the funding fee maxRate.


## Impact
User will close his maker position when he shouldn't be allowed to, and it would cause open taker positions to be greatly impacted. And those who are not actively monitoring their open taker positions will suffer loss due to high funding fees.

## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/collateral/Collateral.sol#L123-L132
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L362-L372
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L334
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L535-L544


## Tool used

Manual Review

## Recommendation
Consider implementing any of these:
- Protocol should receive a share of liquidation fee: This would disincentivize users from wanting to liquidate their own accounts, and they would want to keep their positions healthy and over-collateralized
- Let there be a maker limit on each account: In addition to the global maker limit, there should be maker limit for each account which may be capped at 5% of global maker limit. This would decentralize liquidity provisioning.

