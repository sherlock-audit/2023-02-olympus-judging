shark

medium

# Infinite loop due to use of `continue` before incrementing

## Summary
In the function [`_accumulateExternalRewards`](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L192-L216), an infinite loop will be caused if [line 203](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L203) evaluates to true. This is due to the `continue` keyword being used before `++i` is reached. Because `++i` is never reached, the loop never increments, causing it to continually loop until all gas is used up.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L198-L214

As seen above, the loop is incremented at the end of the loop. However, if `newBalance < rewardToken.lastBalance` were to be true, the loop will not be incremented, since `continue` is used before the loop can increment.  

## Impact
Because of this, a denial of service may happen to all contracts/functions reliant on `_accumulateExternalRewards`.


## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L192-L216
## Tool used

Manual Review

## Recommendation

The following addition will ensure that `i` is properly incremented before using the `continue` keyword

```diff
    function _accumulateExternalRewards() internal override returns (uint256[] memory) {
        uint256 numExternalRewards = externalRewardTokens.length;

        auraPool.rewardsPool.getReward(address(this), true);

        uint256[] memory rewards = new uint256[](numExternalRewards);
        for (uint256 i; i < numExternalRewards; ) {
            ExternalRewardToken storage rewardToken = externalRewardTokens[i];
            uint256 newBalance = ERC20(rewardToken.token).balanceOf(address(this));

            // This shouldn't happen but adding a sanity check in case
            if (newBalance < rewardToken.lastBalance) {
                emit LiquidityVault_ExternalAccumulationError(rewardToken.token);
+               unchecked { ++i; }
                continue;
            }

            rewards[i] = newBalance - rewardToken.lastBalance;
            rewardToken.lastBalance = newBalance;

            unchecked {
                ++i;
            }
        }
        return rewards;
    }
```
