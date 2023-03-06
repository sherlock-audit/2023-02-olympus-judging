rvierdiiev

high

# cachedUserRewards variable is never reset, so user can steal all rewards

## Summary
cachedUserRewards variable is never reset, so user can steal all rewards
## Vulnerability Detail
When user wants to withdraw then `_withdrawUpdateRewardState` function is called.
This function updates internal reward state and claims rewards for user if he provided `true` as `claim_` param.

In case if user didn't want to claim, and `rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]`  then `cachedUserRewards` variable will be set for him which will allow him to claim that amount later.
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L583-L590
```solidity
            if (rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]) {
                userRewardDebts[msg.sender][rewardToken.token] = 0;
                cachedUserRewards[msg.sender][rewardToken.token] +=
                    rewardDebtDiff -
                    userRewardDebts[msg.sender][rewardToken.token];
            } else {
                userRewardDebts[msg.sender][rewardToken.token] -= rewardDebtDiff;
            }
```

When user calls claimRewards, then `cachedUserRewards` variable [is added to the rewards](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L371) he should receive.
The problem is that `cachedUserRewards` variable is never reset to 0, once user claimed that amount.

Because of that he can claim multiple times in order to receive all balance of token.
## Impact
User can steal all rewards
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Once user received rewards, reset `cachedUserRewards` variable to 0. This can be done inside `_claimInternalRewards` function.