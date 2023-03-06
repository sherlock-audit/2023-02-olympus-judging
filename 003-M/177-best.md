0x52

medium

# Reward tokens can never be added again once they are removed without breaking rewards completely

## Summary

Once reward tokens are removed they can never be added back to the contract. The happens because accumulated rewards are tracked differently globally vs individually. Global accumulated rewards are tracked inside the rewardToken array whereas it is tracked by token address for users. When a reward token is removed the global tracker is cleared but the individual trackers are not. If a removed token is added again, the global tracker will reset to zero but the individual tracker won't. As a result of this claiming will fail due to an underflow.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L491-L493

The amount of accumulated rewards for a specific token is tracked in it's respective rewardToken struct. 

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L624-L629

For individual users the rewards are stored in a mapping.

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L694-L703

When a reward token is removed the global tracker for the accumulated rewards is also removed. The problem is that the individual mapping still stores the previously accumulated rewards. If the token is ever added again, the global accumulated reward tracker will now be reset but the individual trackers will not. This will cause an underflow anytime a user tries to claim reward tokens. 

## Impact

Reward tokens cannot be added again once they are removed

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L674-L687

## Tool used

ChatGPT

## Recommendation

Consider tracking accumulatedRewardsPerShare in a mapping rather than in the individual struct or change how removal of reward tokens works