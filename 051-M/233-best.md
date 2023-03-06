minhtrng

medium

# Rewards can accrue beyond the balance of the contract

## Summary

`SingleSidedLiquidityVault` keeps accruing rewards "on paper" even when the contract does not hold enough rewards tokens. This can lead to late claimers not being able to get any rewards.

## Vulnerability Detail

The functions responsible for reward accrual only accrue value on time, but do not check that the contract holds enough reward tokens:

```js
function _accumulateInternalRewards() internal view returns (uint256[] memory) {
    uint256 numInternalRewardTokens = internalRewardTokens.length;
    uint256[] memory accumulatedInternalRewards = new uint256[](numInternalRewardTokens);

    for (uint256 i; i < numInternalRewardTokens; ) {
        InternalRewardToken memory rewardToken = internalRewardTokens[i];

        uint256 totalRewards;
        if (totalLP > 0) {
            //@done-todo where is lastRewardTime updated? -> _updateInternalRewardState
            uint256 timeDiff = block.timestamp - rewardToken.lastRewardTime;
            totalRewards = (timeDiff * rewardToken.rewardsPerSecond);
        }

        accumulatedInternalRewards[i] = totalRewards;

        unchecked {
            ++i;
        }
    }

    return accumulatedInternalRewards;
}
...
function _updateInternalRewardState(uint256 id_, uint256 amountAccumulated_) internal {
    // This correctly uses 1e18 because the LP tokens of all major DEXs have 18 decimals
    InternalRewardToken storage rewardToken = internalRewardTokens[id_];
    if (totalLP != 0)
        rewardToken.accumulatedRewardsPerShare += (amountAccumulated_ * 1e18) / totalLP;
    rewardToken.lastRewardTime = block.timestamp;
}
```

This can lead to a state where users can claim more rewards than the contract holds, causing late claimers to not being able to get rewards.

## Impact

DOS of rewards claiming for some users (effectively a loss of rewards, if the contract does not get topped up with reward tokens)

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/53aff100d0738416b3acaaf3be603c9856bcc661/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L463-L484

https://github.com/sherlock-audit/2023-02-olympus/blob/53aff100d0738416b3acaaf3be603c9856bcc661/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L488-L494

## Tool used

Manual Review

## Recommendation

Keep track of how many rewards can be accrued and limit accrual to that amount.