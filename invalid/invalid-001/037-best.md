Raspy Ebony Whale

medium

# Lack of Safety Check in Maker Deposit Function

## Summary

I have identified a potential security vulnerability in the Maker Deposit function of the D3Funding smart contract within the DODO V3 update. The concern arises from the absence of a thorough safety check before executing the Maker Deposit operation. This could lead to a scenario where funds are added to the contract even in situations where the contract is not deemed secure.

## Vulnerability Detail

The identified vulnerability lies in the makerDeposit function of the D3Funding smart contract within the DODO V3 update. The critical concern arises from the absence of a comprehensive safety check (checkSafe()) before executing the Maker Deposit operation. Below is a detailed breakdown:

## Impact

**Unauthorized Fund Addition:**

**Potential Exploitation:** Without a proper safety check, an attacker could exploit this vulnerability to deposit funds into the contract even in situations where the contract is not considered safe. This could lead to unauthorized fund additions.
Risk of Financial Loss:

**Financial Implications:** If funds are added in unsafe conditions, it poses a risk of financial loss to users of the contract, including liquidity providers and traders, as their assets might be at risk.
System Integrity Compromised:

**Overall System Impact:** The lack of a safety check compromises the integrity of the entire DODO V3 system. It undermines the security guarantees provided by the protocol and could erode user trust.
Potential for Exploitation:

**Exploitation Scenario:** Malicious actors could potentially exploit this vulnerability during market fluctuations or periods of system stress, taking advantage of the lack of safety checks to manipulate the contract's state.

###  Information on Potential Impact on Users and System Integrity:

**Depositing Users:**

**Potential** **Impact**: The absence of safety checks before Maker Deposit creates the possibility of unauthorized fund additions to the contract.

**Consequences**: Depositing users may be exposed to a situation where their assets are added to the contract even in conditions that do not meet security standards. This, in turn, places their capital at potential risk of loss.

**Financial Risk for Users:**

**Potential Impact:** In the case of unsafe fund additions, there is a financial risk for users of the contract, such as liquidity providers or traders.

**Consequences**: Loss of funds by these user groups can negatively impact their trust in the DODO V3 platform and result in significant financial losses.

**Integrity of the DODO V3 System:**

**Potential Impact:** Lack of proper safety checks before depositing may compromise the integrity of the DODO V3 system.

**Consequences:** The system may lose its ability to effectively manage liquidity, affecting the platform's stability. This could lead to transaction difficulties and market manipulation.
Risk of Manipulation During System Stress:

**Potential Impact**: Lack of security before the deposit operation opens a potential loophole for manipulation during periods of market instability or system stress.
**Consequences:** If exploited, there is a risk of contract manipulation, leading to undesired outcomes such as market disarray or system instability.

**Loss of User Trust:**

**Potential Impact**: Security is a key element of trust in a financial platform.

**Consequences**: Lack of proper safety checks before depositing may result in the loss of user trust, impacting the reputation of DODO V3 as a secure and trustworthy financial platform.

**Effects on Market Stability:**

**Potential Impact:** Contract manipulation due to lack of security can impact market stability and fair competition.

**Consequences**: Loss of market stability can lead to negative consequences for all participants, including liquidity providers, traders, and investors.

## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo/blob/main/dodo-v3/contracts/DODOV3MM/D3Pool/D3Funding.sol#L58-L75
`// Inside D3Funding contract
function makerDeposit(address token) external virtual nonReentrant poolOngoing {
    require(ID3Oracle(state._ORACLE_).isFeasible(token), Errors.TOKEN_NOT_FEASIBLE);
    // ... (other code)
    require(checkSafe(), Errors.NOT_SAFE); // Lack of safety check before Maker Deposit
    // ... (other code)
}
`

### Bug Reproduction:
**Initial Steps:**

Ensure that the D3Funding contract is in a state where the lack of safety is apparent.
Verify that the system state is configured to conditions where safety is not met.
Invoke the Maker Deposit Function:

Call the makerDeposit function on the D3Funding contract, passing a specific token as an argument.
Choose a token that would not typically meet the safety conditions according to the checkSafe() function.
Check Safety Approval:

Before implementing the suggested change, check the safety state of the contract using the checkSafe() function.
Note the Lack of Rejection:

Observe that despite the safety conditions not being met, the makerDeposit function is executed without triggering an error.
Pay attention to the fact that the lack of safety does not result in the rejection of the transaction.
## Tool used

Manual Review

## Recommendation
```solidity
// Inside D3Funding contract
function makerDeposit(address token) external virtual nonReentrant poolOngoing {
    require(ID3Oracle(state._ORACLE_).isFeasible(token), Errors.TOKEN_NOT_FEASIBLE);
    
    // Introduce a safety check before Maker Deposit
    require(checkSafeForMakerDeposit(token), Errors.NOT_SAFE_FOR_MAKER_DEPOSIT);

    // ... (other code)
    state.hasDepositedToken[token] = true;
    state.depositedTokenList.push(token);
    // transfer in from proxies
    uint256 tokenInAmount = IERC20(token).balanceOf(address(this)) - state.balances[token];
    _updateReserve(token);
    
    // if token in tokenlist, approve max, ensure vault could force liquidate
    uint256 allowance = IERC20(token).allowance(address(this), state._D3_VAULT_);
    if (_checkTokenInTokenlist(token) && allowance < type(uint256).max) {
        IERC20(token).forceApprove(state._D3_VAULT_, type(uint256).max);
    }
    require(checkSafe(), Errors.NOT_SAFE);

    emit MakerDeposit(token, tokenInAmount);
}

```