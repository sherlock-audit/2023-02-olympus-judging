RaymondFam

medium

# Users repeatedly calling `claimRewards()` will progressively receive higher amount of internal rewards than expected

## Summary
Internal rewards can be claimed via `withdraw()` or `claimRewards()`. However the latter will surprise the user with higher amount of internal rewards than is expected due to a missing update of `rewardToken.lastRewardTime` with `block.timestamp`.

## Vulnerability Detail
In SingleSidedLiquidityVault.sol, `rewardToken.lastRewardTime` is only updated in `_updateInternalRewardState()`. Calling `claimRewards()` is going to skip invoking `_accumulateInternalRewards()` and `_updateInternalRewardState()` which is the standard way of updating `rewardToken.accumulatedRewardsPerShare` in `_withdrawUpdateRewardState()`. Instead, it jumps straight to calling `_claimExternalRewards()` which is going to invoke the last step of the atomic transaction, i.e. `internalRewardsForToken()` to determine the amount of rewards the user has earned. Although the logic of `_accumulateInternalRewards()` and `_updateInternalRewardState()` has been abridged in the if block of `internalRewardsForToken()`, it has missed out a crucial step, i.e. assigning `rewardToken.lastRewardTime` with `block.timestamp`.

## Impact
As a result, the updated `rewardToken.lastRewardTime` will overly inflate `accumulatedRewardsPerShare` in the user's subsequent calls on `claimRewards()` to reward the user with higher than expected amounts of all internal rewards.  

## Code Snippet
Note that `claimRewards()` is going to skip calling `_accumulateInternalRewards()` and `_updateInternalRewardState()` when claiming the user's rewards for all internal reward tokens.

[File: SingleSidedLiquidityVault.sol#L288-L310](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L288-L310)

```solidity
    function claimRewards() external onlyWhileActive nonReentrant {
        uint256 numInternalRewardTokens = internalRewardTokens.length;
        uint256 numExternalRewardTokens = externalRewardTokens.length;

        uint256[] memory accumulatedRewards = _accumulateExternalRewards();

        for (uint256 i; i < numInternalRewardTokens; ) {
            _claimInternalRewards(i);

            unchecked {
                ++i;
            }
        }

        for (uint256 i; i < numExternalRewardTokens; ) {
            _updateExternalRewardState(i, accumulatedRewards[i]);
            _claimExternalRewards(i);

            unchecked {
                ++i;
            }
        }
    }
```
As can be seen in the code block below, the if block condition is going to be true and move on to correctly updating `accumulatedRewardsPerShare` on the first call. All subsequent calls (skipping `withdraw()`) on `claimRewards()` is going to lead to incurring over inflation of `accumulatedRewardsPerShare` because of the same stale `lastRewardTime`.

[File: SingleSidedLiquidityVault.sol#L354-L372](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L354-L372)

```solidity
    function internalRewardsForToken(uint256 id_, address user_) public view returns (uint256) {
        InternalRewardToken memory rewardToken = internalRewardTokens[id_];
        uint256 lastRewardTime = rewardToken.lastRewardTime;
        uint256 accumulatedRewardsPerShare = rewardToken.accumulatedRewardsPerShare;

        if (block.timestamp > lastRewardTime && totalLP != 0) {
            uint256 timeDiff = block.timestamp - lastRewardTime;
            uint256 totalRewards = timeDiff * rewardToken.rewardsPerSecond;

            // This correctly uses 1e18 because the LP tokens of all major DEXs have 18 decimals
            accumulatedRewardsPerShare += (totalRewards * 1e18) / totalLP;
        }

        // This correctly uses 1e18 because the LP tokens of all major DEXs have 18 decimals
        uint256 totalAccumulatedRewards = (lpPositions[user_] * accumulatedRewardsPerShare) -
            userRewardDebts[user_][rewardToken.token];

        return (cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards) / 1e18;
    }
```
## Tool used

Manual Review

## Recommendation
Consider having the affected `internalRewardsForToken()` refactored as follows:

```diff
    function internalRewardsForToken(uint256 id_, address user_) public view returns (uint256) {
        InternalRewardToken memory rewardToken = internalRewardTokens[id_];
        uint256 lastRewardTime = rewardToken.lastRewardTime;
        uint256 accumulatedRewardsPerShare = rewardToken.accumulatedRewardsPerShare;

        if (block.timestamp > lastRewardTime && totalLP != 0) {
            uint256 timeDiff = block.timestamp - lastRewardTime;
            uint256 totalRewards = timeDiff * rewardToken.rewardsPerSecond;

            // This correctly uses 1e18 because the LP tokens of all major DEXs have 18 decimals
            accumulatedRewardsPerShare += (totalRewards * 1e18) / totalLP;
+            rewardToken.lastRewardTime = block.timestamp;
        }

        // This correctly uses 1e18 because the LP tokens of all major DEXs have 18 decimals
        uint256 totalAccumulatedRewards = (lpPositions[user_] * accumulatedRewardsPerShare) -
            userRewardDebts[user_][rewardToken.token];

        return (cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards) / 1e18;
    }
```
This will ensure the next `accumulatedRewardsPerShare` is going to be correctly updated with the accurate `timeDiff` that dictates `totalRewards`.