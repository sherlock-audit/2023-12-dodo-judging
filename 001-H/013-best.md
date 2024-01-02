Ancient Felt Gazelle

high

# Users don't need to pay for swapFee when selling or buying tokens

## Summary

The `D3Trading` contract only charges the `mtFee` and ignores the `swapFee` in function `querySellTokens` and `queryBuyTokens`. This results in users obtaining more `to` tokens with the same amount of `from` tokens or expending fewer `from` tokens to acquire the same amount of `to` tokens.

## Vulnerability Detail

[`queriySellTokens`](https://github.com/sherlock-audit/2023-12-dodo/blob/ea7f786161113144562a900dbff31457ff7025ef/dodo-v3/contracts/DODOV3MM/D3Pool/D3Trading.sol#L206) should return the amount of `to` tokens that user should receive, which should exclude `mtFee` and `swapFee`, but it only returns `receiveToAmount - mtFee`.

```solidity
    function querySellTokens(
        address fromToken,
        address toToken,
        uint256 fromAmount
    ) public view returns (uint256 payFromAmount, uint256 receiveToAmount, uint256 vusdAmount, uint256 swapFee, uint256 mtFee) {
        require(fromAmount > 1000, Errors.AMOUNT_TOO_SMALL);
        Types.RangeOrderState memory D3State = getRangeOrderState(fromToken, toToken);

        {
        uint256 fromTokenDec = IERC20Metadata(fromToken).decimals();
        uint256 toTokenDec = IERC20Metadata(toToken).decimals();
        uint256 fromAmountWithDec18 = Types.parseRealAmount(fromAmount, fromTokenDec);
        uint256 receiveToAmountWithDec18;
        ( , receiveToAmountWithDec18, vusdAmount) =
            PMMRangeOrder.querySellTokens(D3State, fromToken, toToken, fromAmountWithDec18);

        receiveToAmount = Types.parseDec18Amount(receiveToAmountWithDec18, toTokenDec);
        payFromAmount = fromAmount;
        }

        receiveToAmount = receiveToAmount > state.balances[toToken] ? state.balances[toToken] : receiveToAmount;

        uint256 swapFeeRate = D3State.fromTokenMMInfo.swapFeeRate +  D3State.toTokenMMInfo.swapFeeRate;
        swapFee = DecimalMath.mulFloor(receiveToAmount, swapFeeRate);
        uint256 mtFeeRate = D3State.fromTokenMMInfo.mtFeeRate +  D3State.toTokenMMInfo.mtFeeRate;
        mtFee = DecimalMath.mulFloor(receiveToAmount, mtFeeRate);
//@audit -> (receiveToAmount - mtFee) doesn't take the swapFee into consideration, user will receive more to tokens
        return (payFromAmount, receiveToAmount - mtFee, vusdAmount, swapFee, mtFee);
    }
```

The same logic is also implemented in funtion [`queryBuyTokens`](https://github.com/sherlock-audit/2023-12-dodo/blob/ea7f786161113144562a900dbff31457ff7025ef/dodo-v3/contracts/DODOV3MM/D3Pool/D3Trading.sol#L229)

```solidity
    function queryBuyTokens(
        address fromToken,
        address toToken,
        uint256 toAmount
    ) public view returns (uint256 payFromAmount, uint256 receiveToAmount, uint256 vusdAmount, uint256 swapFee, uint256 mtFee) {
        require(toAmount > 1000, Errors.AMOUNT_TOO_SMALL);
        Types.RangeOrderState memory D3State = getRangeOrderState(fromToken, toToken);

        // query amount and transfer out
        uint256 toAmountWithFee;
        {
        uint256 swapFeeRate = D3State.fromTokenMMInfo.swapFeeRate +  D3State.toTokenMMInfo.swapFeeRate;
        swapFee = DecimalMath.mulFloor(toAmount, swapFeeRate);
        uint256 mtFeeRate = D3State.fromTokenMMInfo.mtFeeRate +  D3State.toTokenMMInfo.mtFeeRate;
        mtFee = DecimalMath.mulFloor(toAmount, mtFeeRate);
//@audit -> toAmountWithFee only adds mtFee, swapFee is ignored.
        toAmountWithFee = toAmount + mtFee;
        }

        require(toAmountWithFee <= state.balances[toToken], Errors.BALANCE_NOT_ENOUGH);

        uint256 fromTokenDec = IERC20Metadata(fromToken).decimals();
        uint256 toTokenDec = IERC20Metadata(toToken).decimals();
        uint256 toFeeAmountWithDec18 = Types.parseRealAmount(toAmountWithFee, toTokenDec);
        uint256 payFromAmountWithDec18;
        (payFromAmountWithDec18, , vusdAmount) =
            PMMRangeOrder.queryBuyTokens(D3State, fromToken, toToken, toFeeAmountWithDec18);
        payFromAmount = Types.parseDec18Amount(payFromAmountWithDec18, fromTokenDec);
        if(payFromAmount == 0) {
            payFromAmount = 1;
        }

        return (payFromAmount, toAmount, vusdAmount, swapFee, mtFee);
    }
```

## Impact

Users don't need to pay for `swapFee` when selling or buying tokens

## Code Snippet

https://github.com/sherlock-audit/2023-12-dodo/blob/ea7f786161113144562a900dbff31457ff7025ef/dodo-v3/contracts/DODOV3MM/D3Pool/D3Trading.sol#L206
https://github.com/sherlock-audit/2023-12-dodo/blob/ea7f786161113144562a900dbff31457ff7025ef/dodo-v3/contracts/DODOV3MM/D3Pool/D3Trading.sol#L229

## Tool used

Manual Review

## Recommendation
It is recommended to subtract `swapFee` from `receiveToAmount` in function `querySellTokens` and add `swapFee` to `toAmountWithFee` in function `queryBuyTokens`.