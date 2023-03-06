cducrest-brainbot

medium

# removeInternalRewardToken does not clean storage value for users

## Summary

The function `removeInternalRewardToken()` deletes an internal reward token from the list of reward tokens. However the values of `userRewardDebts` and `cachedUserRewards` for the token are still kept in storage.

## Vulnerability Detail

If the token is added once again in the future, we could easily have inconsistencies with the leftover storage values concerning the users. As an example if the token is added again it will have `accumulatedRewardsPerShare == 0` while some users might have `userRewardDebts` set to a high value. These users may not be able to accumulate rewards in the future for this token.

## Impact

If reward tokens are added / removed / added again, accounting will most likely be broken for some users. This is a problem even if the protocol does not intend to add / remove / add tokens if they unintentionally remove the wrong reward token from a vault, there will be no way for recovery.

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L674-L703

## Tool used

Manual Review

## Recommendation

Let the function `addInternalRewardToken` to provide a value for `accumulatedRewardsPerShare` or loop through user data and delete it (should be fine gas-wise because it's cleaning storage so getting gas refunded).