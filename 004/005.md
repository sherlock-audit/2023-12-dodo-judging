Interesting Sapphire Lizard

medium

# Precision Loss Risk in liquidate Function

## Summary
The `liquidate` function in the provided contract is susceptible to precision loss during the calculation of `collateralAmountMax` due to the sequence of multiplication and division operations. This precision loss may lead to an underestimation of the maximum collateral amount, potentially impacting the accuracy of the liquidation process.
## Vulnerability Detail
In the `liquidate` function, the calculation of `collateralAmountMax` involves multiplying `debtTokenPrice` by `DISCOUNT` and then dividing the result by `collateralTokenPrice`. The sequence of these operations can result in precision loss, particularly if the product of `collateralTokenPrice` and `DISCOUNT` is not an exact multiple of `debtTokenPrice`. The code snippet is as follows:
```solidity
uint256 collateralTokenPrice = ID3Oracle(_ORACLE_).getPrice(collateral);
uint256 debtTokenPrice = ID3Oracle(_ORACLE_).getPrice(debt);
uint256 collateralAmountMax = debtToCover.mul(debtTokenPrice).div(collateralTokenPrice.mul(DISCOUNT));
```
The potential precision loss can impact the accuracy of the liquidation process, leading to incorrect estimations of the maximum collateral amount that can be claimed.
## Impact
The impact of this precision loss is that the liquidation process may not accurately determine the maximum collateral amount that can be claimed. This can result in an underestimation of the collateral required for the liquidation, potentially leading to suboptimal or incorrect outcomes in the liquidation process.
## Code Snippet
[Link](https://github.com/sherlock-audit/2023-12-dodo/blob/main/dodo-v3/contracts/DODOV3MM/D3Vault/D3VaultLiquidation.sol#L48)
## Tool used

Manual Review

## Recommendation
 It is advisable to rearrange the multiplication and division operations to reduce rounding errors.