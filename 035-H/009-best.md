carrot

high

# Incorrect caching of rewards by the `withdraw` function

## Summary
The function `withdraw` in `SingleSidedLiquidityVault.sol` allows for rewards to be cached, if `false` is sent in place of the `bool claim_` argument. However, the math is incorrectly implemented, allowing the user to claim infinite rewards.
## Vulnerability Detail
The function `withdraw` eventually calls the function `_withdrawUpdateRewardState`  to handle reward claims through an if-else block
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L574-L595
If `claim_` is set to false, the function `_claimInternalRewards(i)` is never called. The `rewardDebtDiff` is calculated, and since there are pending rewards for the user, `rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]` must be true, as the first term calculates rewards from the beginning of the contract, while the second term calculates the rewards already paid out. The the code enters the if block.
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L584-L587

These lines are mishandling the maths. The first line sets the `userRewardDebts` to 0. In Masterchef contracts, the `userRewardDebts` variable keeps track of the rewards already paid out to the user. Setting this to 0 lets the user claim rewards which have already been paid out to them.
The second statement has meaningless terms since it is subtracting `userRewardDebts[msg.sender][rewardToken.token]` which has been set to 0 in the previous line. `cachedUserRewards` are rewards the user is eligible to claim, thus setting it to 
`rewardDebtDiff` lets the user claim a large number of rewards.

## Impact
Incorrect caching of rewards leads to users claiming large number of rewards
## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L583-L588
## Tool used

Manual Review

## Recommendation
The variable `userRewardDebts` keeps track of rewards ALREADY paid out, `cachedUserRewards` keeps track of PENDING rewards to be paid, and `rewardDebtDiff` keeps track of all accumulated rewards at this state. The correct implementation would be
```solidity
if (rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]) {
        cachedUserRewards[msg.sender][rewardToken.token] +=
            rewardDebtDiff -
            userRewardDebts[msg.sender][rewardToken.token];
        userRewardDebts[msg.sender][rewardToken.token] += rewardDebtDiff;
}
```
cachedUserRewards must also be flushed and set to 0 whenever the rewards are paid.
This does 2 things: 
1. Cached rewards are set as the reward owed. This is equal to `lp*rewardPerShare - alreadyPaidAmount`
2. user pending rewards are zeroed out. This is done by setting `userRewardDebts` to the current accumulated value.