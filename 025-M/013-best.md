Bauchibred

medium

# Chainlink's L2 Sequencer could be down

## Summary

The Chainlink oracles lack a check to determine if the Arbitrum sequencer is down, when using Chainlink feeds on L2 chains. This omission may result in obtaining prices that appear fresh but are actually outdated, leading to potential vulnerabilities.

## Vulnerability Detail

See summary

## Impact

The impact of this vulnerability depends on the usage of any of the queries but since these queries are going to be used to obtain prices for critical operations, such as determining asset values or executing financial transactions, the lack of sequencer status check would clearly lead to protocol using outdated prices, i.e get better borrows or avoid liquidations

## Code Snippet

Instances of the vulnerable code snippet can be found in [ChainlinkAggregator.sol](https://github.com/sherlock-audit/2023-05-perennial/blob/0f73469508a4cd3d90b382eac2112f012a5a9852/perennial-mono/packages/perennial-oracle/contracts/types/ChainlinkAggregator.sol#L32-L36) and numerous other Chainlink-prefixed contracts.

## Tool used

Manual Review

## Recommendation

It is recommended to follow the code example of Chainlink:
https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code

TLDR: Implement a check to verify the status of the Arbitrum sequencer before retrieving prices from Chainlink feeds as this check should ensure that prices are fetched only when the sequencer is operational and providing up-to-date data.
