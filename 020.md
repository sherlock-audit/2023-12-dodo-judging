Ambitious Holographic Cormorant

medium

# Malicious user can block LPs from providing liquidity to vaults

## Summary

Since the original V3 release the method `withdrawLeft` has been commented out which allowed the owner of the vault to withdraw tokens directly transferred into the vault (that had not been accounted). 

As a result of removing this method it is now possible for a malicious user to grief the protocol by preventing legitimate LPs from contributing to the vault and earning yield.

## Vulnerability Detail

When an LP wants to deposit tokens into a vault they call `userDeposit` in the D3Proxy contract:

```solidity
function userDeposit(address user, address token, uint256 amount, uint256 minDtokenAmount) external payable {
        uint256 dTokenAmount;
        if (token == _ETH_ADDRESS_) {
            require(msg.value == amount, "D3PROXY_PAYMENT_NOT_MATCH");
            _deposit(msg.sender, _D3_VAULT_, _WETH_, amount);
            dTokenAmount = ID3Vault(_D3_VAULT_).userDeposit(user, _WETH_);
        } else {
            _deposit(msg.sender, _D3_VAULT_, token, amount);
            dTokenAmount = ID3Vault(_D3_VAULT_).userDeposit(user, token);
        }
        require(dTokenAmount >= minDtokenAmount, "D3PROXY_MIN_DTOKEN_AMOUNT_FAIL");
    }
```

This makes the underlying call to `userDeposit` in D3VaultFunding:

```solidity
function userDeposit(address user, address token) external nonReentrant allowedToken(token) returns(uint256 dTokenAmount) {
        accrueInterest(token);

        AssetInfo storage info = assetInfo[token];
        uint256 realBalance = IERC20(token).balanceOf(address(this));
        uint256 amount = realBalance  - info.balance;
        require(ID3UserQuota(_USER_QUOTA_).checkQuota(user, token, amount), Errors.EXCEED_QUOTA);
        uint256 exchangeRate = _getExchangeRate(token);
        uint256 totalDToken = IDToken(info.dToken).totalSupply();
        require(totalDToken.mul(exchangeRate) + amount <= info.maxDepositAmount, Errors.EXCEED_MAX_DEPOSIT_AMOUNT);
        dTokenAmount = amount.div(exchangeRate);

        IDToken(info.dToken).mint(user, dTokenAmount);
        info.balance = realBalance;

        emit UserDeposit(user, token, amount, dTokenAmount);
    }
```

As you can see, there is a check here that the `maxDepositAmount` of the vault for a specific token isn’t exceeded. Specifically, `amount` is calculated as the difference between the actual token balance of the vault and the last accounted balance of the vault.

This works fine for normal user interactions, however it opens up the opportunity for a malicious user to grief the protocol by directly transferring tokens into the vault to avoid updating the accounted balance and prevent legitimate LPs from depositing into the vault. The malicious user simply has to transfer a supported token directly to the vault contract (without using the above method) to use up more than the remaining deposit cap.

## Impact

Malicious users can grief the protocol by dumping tokens directly into the vault. This prevents legitimate LPs from contributing to the vault and also reduces the total amount of borrowable supply for pools to borrow from. The end result is a net reduction in value transfer between borrowers and LPs, causing LPs to lose out on potential yield.

## Code Snippet

https://github.com/sherlock-audit/2023-12-dodo/blob/main/dodo-v3/contracts/DODOV3MM/D3Vault/D3Vault.sol#L212-L221

https://github.com/sherlock-audit/2023-12-dodo/blob/main/dodo-v3/contracts/DODOV3MM/periphery/D3Proxy.sol#L157

https://github.com/sherlock-audit/2023-12-dodo/blob/main/dodo-v3/contracts/DODOV3MM/D3Vault/D3VaultFunding.sol#L29-L45

## Tool used

Manual Review

## Recommendation

I’m not sure what the justification was for removing the `withdrawLeft` method since the last audit, but I can’t see why leaving this in would have any negative consequences, maybe I’m missing something here?

Either way I’d definitely recommend having a way to pull excess funds out or alternatively having a method that accounts excess funds as a new deposit to avoid the potential griefing vector.