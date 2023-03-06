cccz

high

# claimFees may cause some external rewards to be locked in the contract

## Summary
claimFees will update rewardToken.lastBalance so that if there are unaccrued reward tokens in the contract, users will not be able to claim them.
## Vulnerability Detail
_accumulateExternalRewards takes the difference between the contract's reward token balance and lastBalance as the reward.
and the accumulated reward tokens are updated by _updateExternalRewardState.
```solidity
    function _accumulateExternalRewards() internal override returns (uint256[] memory) {
        uint256 numExternalRewards = externalRewardTokens.length;

        auraPool.rewardsPool.getReward(address(this), true);

        uint256[] memory rewards = new uint256[](numExternalRewards);
        for (uint256 i; i < numExternalRewards; ) {
            ExternalRewardToken storage rewardToken = externalRewardTokens[i];
            uint256 newBalance = ERC20(rewardToken.token).balanceOf(address(this));

            // This shouldn't happen but adding a sanity check in case
            if (newBalance < rewardToken.lastBalance) {
                emit LiquidityVault_ExternalAccumulationError(rewardToken.token);
                continue;
            }

            rewards[i] = newBalance - rewardToken.lastBalance;
            rewardToken.lastBalance = newBalance;

            unchecked {
                ++i;
            }
        }
        return rewards;
    }
...
    function _updateExternalRewardState(uint256 id_, uint256 amountAccumulated_) internal {
        // This correctly uses 1e18 because the LP tokens of all major DEXs have 18 decimals
        if (totalLP != 0)
            externalRewardTokens[id_].accumulatedRewardsPerShare +=
                (amountAccumulated_ * 1e18) /
                totalLP;
    }

```
auraPool.rewardsPool.getReward can be called by anyone to send the reward tokens to the contract
```solidity
    function getReward(address _account, bool _claimExtras) public updateReward(_account) returns(bool){
        uint256 reward = earned(_account);
        if (reward > 0) {
            rewards[_account] = 0;
            rewardToken.safeTransfer(_account, reward);
            IDeposit(operator).rewardClaimed(pid, _account, reward);
            emit RewardPaid(_account, reward);
        }

        //also get rewards from linked rewards
        if(_claimExtras){
            for(uint i=0; i < extraRewards.length; i++){
                IRewards(extraRewards[i]).getReward(_account);
            }
        }
        return true;
    }
```
However, in claimFees, the rewardToken.lastBalance will be updated to the current contract balance after the admin has claimed the fees.
```solidity
    function claimFees() external onlyRole("liquidityvault_admin") {
        uint256 numInternalRewardTokens = internalRewardTokens.length;
        uint256 numExternalRewardTokens = externalRewardTokens.length;

        for (uint256 i; i < numInternalRewardTokens; ) {
            address rewardToken = internalRewardTokens[i].token;
            uint256 feeToSend = accumulatedFees[rewardToken];

            accumulatedFees[rewardToken] = 0;

            ERC20(rewardToken).safeTransfer(msg.sender, feeToSend);

            unchecked {
                ++i;
            }
        }

        for (uint256 i; i < numExternalRewardTokens; ) {
            ExternalRewardToken storage rewardToken = externalRewardTokens[i];
            uint256 feeToSend = accumulatedFees[rewardToken.token];

            accumulatedFees[rewardToken.token] = 0;

            ERC20(rewardToken.token).safeTransfer(msg.sender, feeToSend);
            rewardToken.lastBalance = ERC20(rewardToken.token).balanceOf(address(this));

            unchecked {
                ++i;
            }
        }
    }
```
Consider the following scenario.
1. Start with rewardToken.lastBalance = 200.
2. After some time, the rewardToken in aura is increased by 100.
3. Someone calls getReward to claim the reward tokens to the contract, and the 100 reward tokens increased have not yet been accumulated via _accumulateExternalRewards and _updateExternalRewardState.
4. The admin calls claimFees to update rewardToken.lastBalance to 290(10 as fees).
5. Users call claimRewards and receives 0 reward tokens. 90 reward tokens will be locked in the contract
## Impact
It will cause some external rewards to be locked in the contract
## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L192-L216
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L496-L503
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L736-L766
## Tool used

Manual Review

## Recommendation
Use _accumulateExternalRewards and _updateExternalRewardState in claimFees to accrue rewards.
```diff
    function claimFees() external onlyRole("liquidityvault_admin") {
        uint256 numInternalRewardTokens = internalRewardTokens.length;
        uint256 numExternalRewardTokens = externalRewardTokens.length;

        for (uint256 i; i < numInternalRewardTokens; ) {
            address rewardToken = internalRewardTokens[i].token;
            uint256 feeToSend = accumulatedFees[rewardToken];

            accumulatedFees[rewardToken] = 0;

            ERC20(rewardToken).safeTransfer(msg.sender, feeToSend);

            unchecked {
                ++i;
            }
        }
+       uint256[] memory accumulatedExternalRewards = _accumulateExternalRewards();
        for (uint256 i; i < numExternalRewardTokens; ) {
+           _updateExternalRewardState(i, accumulatedExternalRewards[i]);
            ExternalRewardToken storage rewardToken = externalRewardTokens[i];
            uint256 feeToSend = accumulatedFees[rewardToken.token];

            accumulatedFees[rewardToken.token] = 0;

            ERC20(rewardToken.token).safeTransfer(msg.sender, feeToSend);
            rewardToken.lastBalance = ERC20(rewardToken.token).balanceOf(address(this));

            unchecked {
                ++i;
            }
        }
    }
```