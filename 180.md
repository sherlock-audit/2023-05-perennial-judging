yixxas

medium

# Chainlink oracle pricing will return a wrong price for asset if underlying aggregator hits minPrice

## Summary
Chainlink oracles have a built in mechanism to prevent erroneous prices. If the price of underlying asset falls below minPrice, it will always return minPrice. In the event of an asset crash, this can be abused as the actual price of asset is considered to be higher by the oracle than what it should be.

## Vulnerability Detail
Chainlink registry calls `latestRoundData` to get the `answer` at some roundId. If the actual price of the asset drops below minPrice, price will be set to minPrice. Leverages that can be made by users depend on their maintenance margin. Users can use this "overpriced asset" as collateral to create a maintenance margin higher than it should be, putting protocol into bad debt. 

```solidity
    function getLatestRound(ChainlinkRegistry self, address base, address quote) internal view returns (ChainlinkRound memory) {
        (uint80 roundId, int256 answer, , uint256 updatedAt, ) =
            FeedRegistryInterface(ChainlinkRegistry.unwrap(self)).latestRoundData(base, quote);
        return ChainlinkRound({roundId: roundId, timestamp: updatedAt, answer: answer});
    }
```

## Impact
In the event of an asset crash, protocol can basically be made bankrupt by abusing an incorrectly priced asset. 

## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-oracle/contracts/types/ChainlinkRegistry.sol#L34-L38

## Tool used

Manual Review

## Recommendation
Consider reverting if returned `answer` hits minPrice or maxPrice as it likely means that `answer` is no longer reliable.
