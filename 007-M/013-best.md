Ruhum

high

# User can receive more rewards through a mistake in the withdrawal logic

## Summary
In the `withdraw()` function of the SingleSidedLiquidityVault the contract updates the reward state. Because of a mistake in the calculation, the user is assigned more rewards than they're supposed to.

## Vulnerability Detail
When a user withdraws their funds, the `_withdrawUpdateRewardState()` function checks how many rewards those LP shares generated. If that amount is higher than the actual amount of reward tokens that the user claimed, the difference between those values is cached and the amount the user claimed is set to 0. That way they receive the remaining shares the next time they claim.

But, the contract resets the number of reward tokens the user claimed *before* it computes the difference. That way, the full amount of reward tokens the LP shares generated are added to the cache.

Here's an example:
1. Alice deposits funds and receives 1e18 shares
2. Alice receives 1e17 rewards and claims those funds immediately
3. Time passes and Alice earns 5e17 more reward tokens
4. Instead of claiming those tokens, Alice withdraws 5e17 (50% of her shares)
That executes `_withdrawUpdateRewardState()` with `lpAmount_ = 5e17` and `claim = false`:
```sol
    function _withdrawUpdateRewardState(uint256 lpAmount_, bool claim_) internal {
        uint256 numInternalRewardTokens = internalRewardTokens.length;
        uint256 numExternalRewardTokens = externalRewardTokens.length;

        // Handles accounting logic for internal and external rewards, harvests external rewards
        uint256[] memory accumulatedInternalRewards = _accumulateInternalRewards();
        uint256[] memory accumulatedExternalRewards = _accumulateExternalRewards();
        for (uint256 i; i < numInternalRewardTokens;) {
            _updateInternalRewardState(i, accumulatedInternalRewards[i]);
            if (claim_) _claimInternalRewards(i);

            // Update reward debts so as to not understate the amount of rewards owed to the user, and push
            // any unclaimed rewards to the user's reward debt so that they can be claimed later
            InternalRewardToken memory rewardToken = internalRewardTokens[i];
            // @audit In our example, rewardDebtDiff = 3e17 (total rewards are 6e17 so 50% of shares earned 50% of reward tokens)
            uint256 rewardDebtDiff = lpAmount_ * rewardToken.accumulatedRewardsPerShare;

            // @audit 3e17 > 1e17
            if (rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]) {

                // @audit userRewardDebts is set to 0 (original value was 1e17, the number of tokens that were already claimed)
                userRewardDebts[msg.sender][rewardToken.token] = 0;
                // @audit cached amount = 3e17 - 0 = 3e17.
                // Alice is assigned 3e17 reward tokens to be distributed the next time they claim
                // The remaining 3e17 LP shares are worth another 3e17 reward tokens.
                // Alice already claimed 1e17 before the withdrawal.
                // Thus, Alice receives 7e17 reward tokens instead of 6e17
                cachedUserRewards[msg.sender][rewardToken.token] +=
                    rewardDebtDiff - userRewardDebts[msg.sender][rewardToken.token];
            } else {
                userRewardDebts[msg.sender][rewardToken.token] -= rewardDebtDiff;
            }

            unchecked {
                ++i;
            }
        }
```

## Impact
A user can receive more reward tokens than they should by abusing the withdrawal system.

## Code Snippet
The issue is that `userRewardDebts` is set to `0` before it's used in the calculation of `cachedUserRewards`: https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L566-L619
```sol
    function _withdrawUpdateRewardState(uint256 lpAmount_, bool claim_) internal {
        uint256 numInternalRewardTokens = internalRewardTokens.length;
        uint256 numExternalRewardTokens = externalRewardTokens.length;

        // Handles accounting logic for internal and external rewards, harvests external rewards
        uint256[] memory accumulatedInternalRewards = _accumulateInternalRewards();
        uint256[] memory accumulatedExternalRewards = _accumulateExternalRewards();

        for (uint256 i; i < numInternalRewardTokens; ) {
            _updateInternalRewardState(i, accumulatedInternalRewards[i]);
            if (claim_) _claimInternalRewards(i);

            // Update reward debts so as to not understate the amount of rewards owed to the user, and push
            // any unclaimed rewards to the user's reward debt so that they can be claimed later
            InternalRewardToken memory rewardToken = internalRewardTokens[i];
            uint256 rewardDebtDiff = lpAmount_ * rewardToken.accumulatedRewardsPerShare;

            if (rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]) {
                userRewardDebts[msg.sender][rewardToken.token] = 0;
                cachedUserRewards[msg.sender][rewardToken.token] +=
                    rewardDebtDiff -
                    userRewardDebts[msg.sender][rewardToken.token];
            } else {
                userRewardDebts[msg.sender][rewardToken.token] -= rewardDebtDiff;
            }

            unchecked {
                ++i;
            }
        }

        for (uint256 i; i < numExternalRewardTokens; ) {
            _updateExternalRewardState(i, accumulatedExternalRewards[i]);
            if (claim_) _claimExternalRewards(i);

            // Update reward debts so as to not understate the amount of rewards owed to the user, and push
            // any unclaimed rewards to the user's reward debt so that they can be claimed later
            ExternalRewardToken memory rewardToken = externalRewardTokens[i];
            uint256 rewardDebtDiff = lpAmount_ * rewardToken.accumulatedRewardsPerShare;

            if (rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]) {
                userRewardDebts[msg.sender][rewardToken.token] = 0;
                cachedUserRewards[msg.sender][rewardToken.token] +=
                    rewardDebtDiff -
                    userRewardDebts[msg.sender][rewardToken.token];
            } else {
                userRewardDebts[msg.sender][rewardToken.token] -= rewardDebtDiff;
            }

            unchecked {
                ++i;
            }
        }
    }

```

## Tool used

Manual Review

## Recommendation
First calculate `cachedUserRewards` then reset `userRewardDebts`.
