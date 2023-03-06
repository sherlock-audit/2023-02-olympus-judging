rvierdiiev

medium

# SingleSidedLiquidityVault._accumulateInternalRewards will revert with underflow error if rewardToken.lastRewardTime is bigger than current time

## Summary
SingleSidedLiquidityVault._accumulateInternalRewards will revert with underflow error if rewardToken.lastRewardTime is bigger than current time
## Vulnerability Detail
Function `_accumulateInternalRewards` is used by almost all external function of SingleSidedLiquidityVault.
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L463-L484
```solidity
    function _accumulateInternalRewards() internal view returns (uint256[] memory) {
        uint256 numInternalRewardTokens = internalRewardTokens.length;
        uint256[] memory accumulatedInternalRewards = new uint256[](numInternalRewardTokens);


        for (uint256 i; i < numInternalRewardTokens; ) {
            InternalRewardToken memory rewardToken = internalRewardTokens[i];


            uint256 totalRewards;
            if (totalLP > 0) {
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
```
The line is needed to see is this `uint256 timeDiff = block.timestamp - rewardToken.lastRewardTime`. In case if `rewardToken.lastRewardTime > block.timestamp` than function will revert and ddos functions that use it.

This is how this can happen.
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L674-L688
```solidity
    function addInternalRewardToken(
        address token_,
        uint256 rewardsPerSecond_,
        uint256 startTimestamp_
    ) external onlyRole("liquidityvault_admin") {
        InternalRewardToken memory newInternalRewardToken = InternalRewardToken({
            token: token_,
            decimalsAdjustment: 10**ERC20(token_).decimals(),
            rewardsPerSecond: rewardsPerSecond_,
            lastRewardTime: block.timestamp > startTimestamp_ ? block.timestamp : startTimestamp_,
            accumulatedRewardsPerShare: 0
        });


        internalRewardTokens.push(newInternalRewardToken);
    }
```
In case if `startTimestamp_` is in the future, then it will be set and cause that problem.
`lastRewardTime: block.timestamp > startTimestamp_ ? block.timestamp : startTimestamp_`.

Now till, `startTimestamp_` time,  `_accumulateInternalRewards` will not work, so vault will be stopped. 
And of course, admin can remove that token and everything will be fine. That's why i think this is medium.
## Impact
SingleSidedLiquidityVault will be blocked
## Code Snippet
Provided above.
## Tool used

Manual Review

## Recommendation
Skip token if it's `lastRewardTime` is in future.