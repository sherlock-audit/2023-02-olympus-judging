GimelSec

medium

# A growing ohmRemoved may cause _canDeposit revert.

## Summary

As time passes, `ohmRemoved` keeps growing. The sanity check in `_canDeposit` could revert if `LIMIT + ohmRemoved` overflows. 

## Vulnerability Detail

As time passes, `ohmRemoved` keeps growing. There is no way to decrease `ohmRemoved`.
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L277
```solidity
    function withdraw(
        uint256 lpAmount_,
        uint256[] calldata minTokenAmounts_,
        bool claim_
    ) external onlyWhileActive nonReentrant returns (uint256) {
        …
        ohmRemoved += ohmReceived > ohmMinted ? ohmReceived - ohmMinted : 0;

        …
    }
```
The sanity check in `_canDeposit` could revert if `LIMIT + ohmRemoved` overflows. 
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L407
```solidity
    function _canDeposit(uint256 amount_) internal view virtual returns (bool) {
        if (amount_ + ohmMinted > LIMIT + ohmRemoved) revert LiquidityVault_LimitViolation();
        return true;
    }
```


## Impact

`_canDeposit` could be DoSed. But it can be fixed if the admin sets a lower `LIMIT`. So we consider it a medium issue.

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L277
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L407

## Tool used

Manual Review

## Recommendation

```solidity
    function _canDeposit(uint256 amount_) internal view virtual returns (bool) {
        if (amount_ + ohmMinted > LIMIT && amount_ + ohmMinted - LIMIT  > ohmRemoved) revert LiquidityVault_LimitViolation();
        return true;
    }
```
