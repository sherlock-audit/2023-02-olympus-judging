HonorLt

medium

# Functions and variables with unclear intentions

## Summary

## Vulnerability Detail

There are certain parts of the codebase that are not entirely correct or have no clear usage:

1) `lpAmountOut` is never assigned a value and thus will always return 0:
```solidity
    function deposit(uint256 amount_, uint256 slippageParam_)
        external
        onlyWhileActive
        nonReentrant
        returns (uint256 lpAmountOut)
```

2)  An internal function that is not used anywhere and thus practically inaccessible:
```solidity
    function getUserWstethShare(address user_) internal view returns (uint256) 
```

3) `decimalsAdjustment` is not used in any meaningful way later:
```solidity
    function addInternalRewardToken(
        address token_,
        uint256 rewardsPerSecond_,
        uint256 startTimestamp_
    ) external onlyRole("liquidityvault_admin") {
        InternalRewardToken memory newInternalRewardToken = InternalRewardToken({
            token: token_,
            decimalsAdjustment: 10**ERC20(token_).decimals(),
            ...
        });
```
```solidity
    function addExternalRewardToken(address token_) external onlyRole("liquidityvault_admin") {
        ExternalRewardToken memory newRewardToken = ExternalRewardToken({
            token: token_,
            decimalsAdjustment: 10**ERC20(token_).decimals(),
            ...
    }
```

## Impact

It does not cause direct harm but might lead to misbehaving under certain conditions. It is hard to tell the exact impact without knowing the intentions.

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L191

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L326-L336

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L51

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L59

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L681

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L711

## Tool used

Manual Review

## Recommendation

Consider fixing the aforementioned cases based on what was intended.
