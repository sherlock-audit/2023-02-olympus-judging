tsvetanovv

medium

# Solmate safeTransfer and safeTansferFrom does not check the code size of the token address

## Summary
The `safeTransfer` and `safeTransferFrom` don't check the existence of code at the token address. This is a known issue while using solmate's libraries. Hence this may lead to miscalculation of funds and may lead to loss of funds, because if `safeTransfer()` and `safeTransferFrom()` are called on a token address that doesn't have a contract in it, it will always return success, bypassing the return value check.

## Vulnerability Detail
See Summary

## Impact
Due to this protocol will think that funds have been transferred successfully, and records will be accordingly calculated, but in reality, funds were never transferred. So this will lead to miscalculation and possibly loss of funds.

## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L216

```solidity
216: pairToken.safeTransferFrom(msg.sender, address(this), amount_);
```

## Tool used

Manual Review

## Recommendation

Use openzeppelin's safeERC20 or implement a code existence check.