mahdikarimi

medium

# first depositor can claim rewards before startTimestamp_

## Summary
If an internal token startTimestamp_ ( The timestamp at which to start distributing rewards ) is higher that block.timestamp first depositor can earn rewards before startTimestamp_  .
## Vulnerability Detail
Let's say the current timestamp is 10 and lastRewardTime is 15 ( when add new internal token startTimestamp_  sets as lastRewardTime  ) , it means distributing rewards should start from timestamp of 15 , now if first depositor deposit , the _updateInternalRewardState function will set lastRewardTime to block.timestamp which cause to start earning rewards from this timestamp . 
```solidity
 function _updateInternalRewardState(uint256 id_, uint256 amountAccumulated_) internal {
        // This correctly uses 1e18 because the LP tokens of all major DEXs have 18 decimals
        InternalRewardToken storage rewardToken = internalRewardTokens[id_];
        if (totalLP != 0)
            rewardToken.accumulatedRewardsPerShare += (amountAccumulated_ * 1e18) / totalLP;
        rewardToken.lastRewardTime = block.timestamp;
    }

```
## Impact
first depositor can claim rewards before startTimestamp_ or token starts to distributing rewards before startTimestamp_  . 
## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L488-L494
## Tool used

Manual Review

## Recommendation
add a check to ensure block.timestamp is larger than lastRewardTime . 