GimelSec

high

# `cachedUserRewards` and `userRewardDebts` shouldn’t be divided by 1e18 in `internalRewardsForToken()` and `externalRewardsForToken()

## Summary

`cachedUserRewards` and `userRewardDebts` shouldn’t be divided by 1e18 in `internalRewardsForToken()` and `externalRewardsForToken()`. since `cachedUserRewards` and `userRewardDebts` share the same precision and the returned values of `internalRewardsForToken()` and `externalRewardsForToken()` will be pushed back to `userRewardDebts`

## Vulnerability Detail

In `_withdrawUpdateRewardState`, the unclaimed reward(cachedUserRewards) is ` rewardDebtDiff - userRewardDebts[msg.sender][rewardToken.token]`. And `rewardDebtDiff` is `lpAmount_ * rewardToken.accumulatedRewardsPerShare`. Thus, we can know that `cachedUserRewards` and `userRewardDebts` should share the same precision.
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L584-L587
```solidity
    function _withdrawUpdateRewardState(uint256 lpAmount_, bool claim_) internal {
        …

            // Update reward debts so as to not understate the amount of rewards owed to the user, and push
            // any unclaimed rewards to the user's reward debt so that they can be claimed later
            InternalRewardToken memory rewardToken = internalRewardTokens[i];
            uint256 rewardDebtDiff = lpAmount_ * rewardToken.accumulatedRewardsPerShare;

            if (rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]) {

                userRewardDebts[msg.sender][rewardToken.token] = 0;
                cachedUserRewards[msg.sender][rewardToken.token] +=
                    rewardDebtDiff -
                    userRewardDebts[msg.sender][rewardToken.token];

        …

            if (rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]) {
                userRewardDebts[msg.sender][rewardToken.token] = 0;
                cachedUserRewards[msg.sender][rewardToken.token] +=
                    rewardDebtDiff -
                    userRewardDebts[msg.sender][rewardToken.token];
        …
        }
    }
```

In `_claimInternalRewards` and `_claimExternalRewards`, the returned value of `internalRewardsForToken` and `externalRewardsForToken` are pushed back to `userRewardDebts`.
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L623
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L636
```solidity
    function _claimInternalRewards(uint256 id_) internal {
        address rewardToken = internalRewardTokens[id_].token;
        uint256 reward = internalRewardsForToken(id_, msg.sender);
        uint256 fee = (reward * FEE) / PRECISION;

        userRewardDebts[msg.sender][rewardToken] += reward;
        accumulatedFees[rewardToken] += fee;

        if (reward > 0) ERC20(rewardToken).safeTransfer(msg.sender, reward - fee);

        emit RewardsClaimed(msg.sender, rewardToken, reward - fee);
    }

    function _claimExternalRewards(uint256 id_) internal {
        ExternalRewardToken storage rewardToken = externalRewardTokens[id_];
        uint256 reward = externalRewardsForToken(id_, msg.sender);
        uint256 fee = (reward * FEE) / PRECISION;

        userRewardDebts[msg.sender][rewardToken.token] += reward;
        accumulatedFees[rewardToken.token] += fee;

        if (reward > 0) ERC20(rewardToken.token).safeTransfer(msg.sender, reward - fee);
        rewardToken.lastBalance = ERC20(rewardToken.token).balanceOf(address(this));

        emit RewardsClaimed(msg.sender, rewardToken.token, reward - fee);
    }
```

In `internalRewardsForToken` and `externalRewardsForToken`, the calculation of returned value is `(cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards) / 1e18`
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L371
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L385
```solidity
    function internalRewardsForToken(uint256 id_, address user_) public view returns (uint256) {
        …
        uint256 totalAccumulatedRewards = (lpPositions[user_] * accumulatedRewardsPerShare) -
            userRewardDebts[user_][rewardToken.token];

        return (cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards) / 1e18;
    }

    function externalRewardsForToken(uint256 id_, address user_) public view returns (uint256) {
        …
        uint256 totalAccumulatedRewards = (lpPositions[user_] *
            rewardToken.accumulatedRewardsPerShare) - userRewardDebts[user_][rewardToken.token];

        return (cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards) / 1e18;
    }
```

In conclusion, the calculation in `_claimExternalRewards` and `_claimExternalRewards` is:
```solidity
uint256 reward = internalRewardsForToken()
= (cachedUserRewards + totalAccumulatedRewards) / 1e18
= (cachedUserRewards + lpPositions * rewardToken.accumulatedRewardsPerShare - userRewardDebts) / 1e18;

userRewardDebts += reward
=> userRewardDebts += (cachedUserRewards + lpPositions * rewardToken.accumulatedRewardsPerShare - userRewardDebts) / 1e18
```

The correct calculation should be:
```solidity
userRewardDebts += cachedUserRewards + (lpPositions * rewardToken.accumulatedRewardsPerShare / 1e18) - userRewardDebts 
```

## Impact

Wrong calculation leads to accounting error. and it could destroy the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L371
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L385
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L584-L587
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L623
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L636



## Tool used

Manual Review

## Recommendation

Fix the calculation in `internalRewardsForToken` and ``
```solidity
    function internalRewardsForToken(uint256 id_, address user_) public view returns (uint256) {
        …
        uint256 totalAccumulatedRewards = (lpPositions[user_] * accumulatedRewardsPerShare) / 1e18 -
            userRewardDebts[user_][rewardToken.token];

        return cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards;
    }

    function externalRewardsForToken(uint256 id_, address user_) public view returns (uint256) {
        …
        uint256 totalAccumulatedRewards = (lpPositions[user_] *
            rewardToken.accumulatedRewardsPerShare)  / 1e18 - userRewardDebts[user_][rewardToken.token];

        return cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards;
    }
```

But there is another issue(Faulty math in `internalRewardsForToken()` leads to Denial of Service) lying in `internalRewardsForToken()` and `externalRewardsForToken()`. The following modification can fix both issues:
```solidity
    function internalRewardsForToken(uint256 id_, address user_) public view returns (uint256) {
        …
        uint256 totalAccumulatedRewards = cachedUserRewards[user_][rewardToken.token] + (lpPositions[user_] * accumulatedRewardsPerShare) / 1e18 -
            userRewardDebts[user_][rewardToken.token];

        return totalAccumulatedRewards;
    }

    function externalRewardsForToken(uint256 id_, address user_) public view returns (uint256) {
        …
        uint256 totalAccumulatedRewards = cachedUserRewards[user_][rewardToken.token] + (lpPositions[user_] *
            rewardToken.accumulatedRewardsPerShare)  / 1e18 - userRewardDebts[user_][rewardToken.token];

        return totalAccumulatedRewards;
    }
```
