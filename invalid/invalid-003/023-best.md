Narrow Ivory Mole

medium

# D3Oracle will use the wrong price if the Chainlink returns price outside normal range

## Summary
ChainlinkAggregators have minPrice and maxPrice circuit breakers built into them. This means that if the price of the asset drops below the minPrice, the protocol will continue to value the token at minPrice instead of it's actual value and the other way round. This will allow users to take out huge amounts of bad debt.

## Vulnerability Detail
The `getPriceFromFeed` function should check for the min and max amount return to prevent cases like LUNA in which the Oracle will return the minimum price and not the crashed price. This would allow user to executes transactions with the asset but at the wrong price. This is exactly what happened to Venus on BSC when LUNA imploded.
## Impact
In an event of extreme asset colatility, the price gotten will be the wrong and not actual price.
## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo/blob/ea7f786161113144562a900dbff31457ff7025ef/dodo-v3/contracts/DODOV3MM/periphery/D3Oracle.sol#L115C4-L124C6
```solidity
    function getPriceFromFeed(address token) internal view returns (uint256) {
        checkSequencerActive();
        require(priceSources[token].isWhitelisted, "INVALID_TOKEN");
        AggregatorV3Interface priceFeed = AggregatorV3Interface(priceSources[token].oracle);
        (uint80 roundID, int256 price,, uint256 updatedAt, uint80 answeredInRound) = priceFeed.latestRoundData();
        require(price > 0, "Chainlink: Incorrect Price");
        require(block.timestamp - updatedAt < priceSources[token].heartBeat, "Chainlink: Stale Price");
        require(answeredInRound >= roundID, "Chainlink: Stale Price");
        return uint256(price);
    }
```
## Tool used
Manual Code Review

## Recommendation

Some check like this can be added to avoid returning of the min price or the max price in case of the price crashes.
```solidity
          require(price < _maxPrice, "Upper price bound");
          require(price > _minPrice, "Lower price bound");
```