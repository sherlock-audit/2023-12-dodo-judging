Savory Magenta Antelope

high

# The checkBorrowSafe method is not set properly

## Summary
according to the DODO Docs, `Borrowing Collateral Ratio` serves as an additional check condition when SP borrows funds. After borrowing, it must be ensured that `borrowing collateral ratio` is greater than 1 + IM.
https://docs.dodoex.io/en/product/dodo-v3-pools/funding-model#b--object-object

## Vulnerability Detail
https://github.com/sherlock-audit/2023-12-dodo/blob/ea7f786161113144562a900dbff31457ff7025ef/dodo-v3/contracts/DODOV3MM/D3Vault/D3VaultFunding.sol#L304
However, in the `checkBorrowSafe`  method， the `Borrowing Collateral Ratio` is greater than IM.

For simplicity, assume the following scenario:

IM=40%
now pool has,
balance: 300 token3
borrowed: 200 token3
maxCollateralAmount = 100
min(maxCollateralAmount, balance - borrowed) = 100

at this point,
collateralRatioBorrow = 100 / 200 = 50% > IM
total margin value==100
total borrowed value==200,
Which means that` total margin value`(how much SP has deposited) can be less than `total borrowed value`.

```solidity
  function testpoc() public {
      
        // token3 price: $1

        
        vm.prank(user1);
        token3.approve(address(dodoApprove), type(uint256).max);
        mockUserQuota.setUserQuota(user1, address(token3), 1000 ether);
        
        vm.prank(user1);
        d3Proxy.userDeposit(user1, address(token3), 200 ether, 0);

     
        token3.mint(address(d3MM), 100 ether);
        d3MM.updateReserve(address(token3));

        poolBorrow(address(d3MM), address(token3), 200 ether);


        logCollateralRatio(address(d3MM));
    }
Logs:
 collateralRatio 11579208923731619542357098500868790785326998466564056403945758 %
 collateralRatioBorrow 50 %
```



## Impact
A malicious SP from borrowing funds indefinitely and causing other SPs to have no available funds.

## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo/blob/ea7f786161113144562a900dbff31457ff7025ef/dodo-v3/contracts/DODOV3MM/D3Vault/D3VaultFunding.sol#L304

## Tool used

Manual Review

## Recommendation
Please confirm whether it is doc issue or a design choices.
