Decent Gauze Goldfish

high

# `payFromAmount` is calculated with incorrect fee amount

## Summary
This protocol has two different fees: swap fee and the maintainer fee. The protocol calculates incorrect `payFromAmount` in the `queryBuyTokens()` function. It uses maintainer fee instead of the swap fee for this calculation.

## Vulnerability Detail
Users provide exactly how many `toToken` to buy in `D3Trading::buyToken()` function, and pay the necessary amount of `fromToken`. The `fromToken` amount to pay is queried with the `D3Trading::queryBuyTokens` function. Another thing to mention is that the swap fees are normally paid as `toToken`s in this protocol.

The function flow for `buyToken()` is like this:

1. User provides exactly how many `toToken` to buy (`toAmount`).
    
2. `queryBuyTokens` function adds the fee and gets `toAmountWithFee`.
    
3. Calculates how many from token is needed to be paid using this "fee added" `toToken` amount.
    
4. `queryBuyTokens` function returns the necessary `payFromAmount`(*newly calculated*) and the `toAmount`(*user provided - exact same amount*).
    
5. `buyToken()` function performs transfers and callback, and then records the swap with `_recordSwap()`.
    

The issue in the codebase is in the second step above. Incorrect fees are added for this calculation. The function adds the maintainer fee (`mtFee`) instead of swap fee (`swapFee`).

Here is the [buyToken() function](https://github.com/sherlock-audit/2023-12-dodo/blob/main/dodo-v3/contracts/DODOV3MM/D3Pool/D3Trading.sol#L133C4-L172C6):

```solidity
    function buyToken(
        address to,
        address fromToken,
        address toToken,
        uint256 quoteAmount,
        uint256 maxPayAmount,
        bytes calldata data
    ) external virtual poolOngoing nonReentrant returns (uint256) {
    // skipped for brevity

        // query amount and transfer out
-->     (uint256 payFromAmount, uint256 receiveToAmount, uint256 vusdAmount, uint256 swapFee, uint256 mtFee) = //@audit payFromAmount here is incorrectly calculated.
-->         queryBuyTokens(fromToken, toToken, quoteAmount);
        require(payFromAmount <= maxPayAmount, Errors.MAXPAY_NOT_ENOUGH);

    // skipped for brevity. Transfers and callback

        // record swap
        uint256 toTokenDec = IERC20Metadata(toToken).decimals();
-->     _recordSwap(fromToken, toToken, vusdAmount, Types.parseRealAmount(receiveToAmount + swapFee, toTokenDec)); //@audit recordSwap uses "receiveToAmaount + swapFee". There will be a mismatch
        require(checkSafe(), Errors.BELOW_IM_RATIO);

        emit Swap(to, fromToken, toToken, payFromAmount, receiveToAmount, swapFee, mtFee, 1);
        return payFromAmount;
    }
```

It queries, makes transfers and records the swap.

And here is the [queryBuyTokens() function](https://github.com/sherlock-audit/2023-12-dodo/blob/main/dodo-v3/contracts/DODOV3MM/D3Pool/D3Trading.sol#L214C1-L246C6):

```solidity
    function queryBuyTokens(
        address fromToken,
        address toToken,
        uint256 toAmount
    ) public view returns (uint256 payFromAmount, uint256 receiveToAmount, uint256 vusdAmount, uint256 swapFee, uint256 mtFee) {
        // skipped for brevity

        // query amount and transfer out
        uint256 toAmountWithFee;
        {
        uint256 swapFeeRate = D3State.fromTokenMMInfo.swapFeeRate +  D3State.toTokenMMInfo.swapFeeRate;
        swapFee = DecimalMath.mulFloor(toAmount, swapFeeRate);
        uint256 mtFeeRate = D3State.fromTokenMMInfo.mtFeeRate +  D3State.toTokenMMInfo.mtFeeRate;
        mtFee = DecimalMath.mulFloor(toAmount, mtFeeRate);
-->     toAmountWithFee = toAmount + mtFee; //@audit both swapFee and mtFee is calculated above but swapFee is not used here. "toAmountWithFee" includes the maintainer fee instead of swap fee.
        }

        require(toAmountWithFee <= state.balances[toToken], Errors.BALANCE_NOT_ENOUGH);

        uint256 fromTokenDec = IERC20Metadata(fromToken).decimals();
        uint256 toTokenDec = IERC20Metadata(toToken).decimals();
        uint256 toFeeAmountWithDec18 = Types.parseRealAmount(toAmountWithFee, toTokenDec);
        uint256 payFromAmountWithDec18;
        (payFromAmountWithDec18, , vusdAmount) =
-->         PMMRangeOrder.queryBuyTokens(D3State, fromToken, toToken, toFeeAmountWithDec18); //@audit payFromAmount is calculated with the fee added toAmount but the fee added to amount is incorrect.
-->     payFromAmount = Types.parseDec18Amount(payFromAmountWithDec18, fromTokenDec);
        if(payFromAmount == 0) {
            payFromAmount = 1;
        }

        return (payFromAmount, toAmount, vusdAmount, swapFee, mtFee);
    }
```

As we can see above, `toAmountWithFee = toAmount + mtFee`, which is incorrect. It should have been the `toAmount + swapFee`, or `toAmount + swapFee + mtFee` if the protocol expects the user to pay both fees.

We can see this in the protocol's previous contest codebase. payFromAmount is calculated with `swapFee`.  
[https://github.com/sherlock-audit/2023-06-dodo/blob/a8d30e611acc9762029f8756d6a5b81825faf348/new-dodo-v3/contracts/DODOV3MM/D3Pool/D3Trading.sol#L212](https://github.com/sherlock-audit/2023-06-dodo/blob/a8d30e611acc9762029f8756d6a5b81825faf348/new-dodo-v3/contracts/DODOV3MM/D3Pool/D3Trading.sol#L212)

```solidity
// previous contest codebase.
        toAmount += swapFee;
```

This might have multiple impacts.

1. If the protocol's swap fee rate is greater than the maintainer fee rate at the time of the swap, user will pay less `fromToken`.
    
2. If the swap fee rate is smaller than the maintainer fee rate, user will have to pay more `fromToken`.
    
3. If the swap fee rate and maintainer fee rate is different, regardless of which one is bigger, `_recordSwap` function will be called with [incorrect value](https://github.com/sherlock-audit/2023-12-dodo/blob/main/dodo-v3/contracts/DODOV3MM/D3Pool/D3Trading.sol#L167C75-L167C100) since it is calculated with `mtFee`, not with the `swapFee`.

## Impact

- User pays more than required or less than required from tokens depending on the difference between maintainer fee and swap fee.
- `_recordSwap` function is called with incorrect value leading incorrect state updates.

## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo/blob/main/dodo-v3/contracts/DODOV3MM/D3Pool/D3Trading.sol#L229

## Tool used

Manual Review

## Recommendation
Consider using `swapFee` instead of `mtFee` while calculating `payFromAmount`. 
If the protocol expects users to pay both fees, then both of them should be added during the calculation.
