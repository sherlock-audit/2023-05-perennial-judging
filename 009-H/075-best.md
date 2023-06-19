cergyk

high

# Malicious trader can bypass utilization buffer

## Summary
A malicious trader can bypass the Utilisation buffer, and push utilisation to 1 on any product.

## Vulnerability Detail
Perenial products use a utilization buffer to prevent taker side to DOS maker by taking up all the liquidity:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L547-L556

Makers would not be able to withdraw if utilization would reach 100% because of `takerInvariant` which is enforced during `closeMake`:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L535-L544

A malicious trader can bypass the utilisation buffer by first opening a maker position, open taker position and close previous maker position.
The only constraint is that she has to use different accounts to open the maker positions and the taker positions, since perennial doesn't allow to have maker and taker positions on a product simultaneously.

So here's a concrete example:

Let's say the state of product is `900 USD` on the maker side, `800 USD` on the taker side, which respects the 10% utilization buffer.

### Example
>Using account 1, Alice opens up a maker position for `100 USD`, bringing maker total to `1000 USD`.

>Using account 2, Alice can open a taker position for `100 USD`, bringing taker to `900 USD`, which respects the 10% utilization buffer still.

>Now using account 1 again, Alice can close her `100 USD` maker position and withdraw collateral, clearing account 1 on perennial completely.

>This brings the utilization to `100%`, since taker = maker = `900 USD`. 

>This is allowed, since only `takerInvariant` is checked when closing a maker position, which enforces that utilization ratio is lower than or equal to `100%`.

## Impact
Any trader can bring the utilization up to 100%, and use that to DoS withdrawals from Products and Balanced vaults for an indefinite amount of time.
This is especially critical for vaults, since when any product related to any market is fully utilized, all redeems from the balanced vault are blocked.

## Code Snippet

## Tool used

Manual Review

## Recommendation
If a safety buffer needs to be enforced for the utilisation of products, it needs to be enforced on closing make positions as well.