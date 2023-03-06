cccz

high

# The contract does not decrease cachedUserRewards but directly increases userRewardDebts when the user claims the reward, which will result in an overflow in internalRewardsForToken/externalRewardsForToken when the user claims the reward next time.

## Summary
The contract does not decrease cachedUserRewards but directly increases userRewardDebts when the user claims the reward, which will result in an overflow in internalRewardsForToken/externalRewardsForToken when the user claims the reward next time.
## Vulnerability Detail
cachedUserRewards will keep the user's unclaimed rewards.
However, when the user claims the reward, instead of decreasing the cachedUserRewards, the userRewardDebts are directly increased, which causes the user's userRewardDebts to be too large.
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
When the user claims the reward again, the large userRewardDebts in internalRewardsForToken/externalRewardsForToken may cause an overflow and prevent the user from claiming the reward.
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
            userRewardDebts[user_][rewardToken.token];                                             // @audit:  overflow here

        return (cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards) / 1e18;
    }

    /// @notice                         Returns the amount of rewards a user has earned for a given external reward token
    /// @param  id_                     The ID of the external reward token
    /// @param  user_                   The user's address to check rewards for
    /// @return uint256                 The amount of rewards the user has earned
    function externalRewardsForToken(uint256 id_, address user_) public view returns (uint256) {
        ExternalRewardToken memory rewardToken = externalRewardTokens[id_];

        // This correctly uses 1e18 because the LP tokens of all major DEXs have 18 decimals
        uint256 totalAccumulatedRewards = (lpPositions[user_] *
            rewardToken.accumulatedRewardsPerShare) - userRewardDebts[user_][rewardToken.token];  // @audit:  overflow here

        return (cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards) / 1e18;
    }
```
Consider the following scenario
1. cachedUserRewards[alice] == 1000, alice calls claimRewards to claim the reward, then cachedUserRewards[alice] == 1000, userRewardDebts[alice] == 1000.
2. at this point accumulatedRewardsPerShare = 100, alice is deposited 2 lp, cachedUserRewards[alice] = 1000+100 * 2 = 1200
3. accumulatedRewardsPerShare increases to 200, alice calls claimRewards to claim rewards. Since 2 * 200 - 1200 will overflow, alice cannot claim the reward

Running this POC requires fixing another vulnerability first.
```diff
            if (rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]) {
-               userRewardDebts[msg.sender][rewardToken.token] = 0;
                cachedUserRewards[msg.sender][rewardToken.token] +=
                    rewardDebtDiff -
                    userRewardDebts[msg.sender][rewardToken.token];
+               userRewardDebts[msg.sender][rewardToken.token] = 0;
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
-               userRewardDebts[msg.sender][rewardToken.token] = 0;
                cachedUserRewards[msg.sender][rewardToken.token] +=
                    rewardDebtDiff -
                    userRewardDebts[msg.sender][rewardToken.token];
+               userRewardDebts[msg.sender][rewardToken.token] = 0;
            } else {
                userRewardDebts[msg.sender][rewardToken.token] -= rewardDebtDiff;
            }
```
POC is as follows. 
```solidity
    function test_POC(address user_) public {
        vm.assume(user_ != address(0) && user_ != alice && user_ != address(liquidityVault));

        // Setup
        _withdrawTimeSetUp();

        // Add second depositor
        vm.startPrank(user_);
        wsteth.mint(user_, 1e18);
        wsteth.approve(address(liquidityVault), 1e18);
        liquidityVault.deposit(1e18, 1e18);
        vm.stopPrank();
        vm.warp(block.timestamp + 10); // Increase time by 10 seconds

        // Alice's rewards should be 15 REWARD tokens and 5 REWARD2 token
        // User's rewards should be 5 REWARD tokens and 5 REWARD2 tokens
        assertEq(liquidityVault.internalRewardsForToken(0, alice), 15e18);
        assertEq(liquidityVault.internalRewardsForToken(0, user_), 5e18);
        vm.prank(user_);
        liquidityVault.withdraw(1e18, minTokenAmounts_, false);
        assertEq(liquidityVault.internalRewardsForToken(0, user_), 5e18);
        vm.prank(user_);
        liquidityVault.claimRewards();
        vm.prank(user_);
        liquidityVault.claimRewards();   // overflow here
    }
```
The output
```sh
Failing tests:
Encountered 1 failing test in src/test/policies/lending/WstethLiquidityVaultMock.t.sol:WstethLiquidityVaultTest
[FAIL. Reason: Arithmetic over/underflow Counterexample: calldata=0x7156fa2f0000000000000000000000000000000000000000000000000000000000000001, args=[0x0000000000000000000000000000000000000001]]
```
## Impact
It may prevent the user from claiming the reward.
## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L354-L386
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L625-L650
## Tool used

Manual Review

## Recommendation
In _claimInternalRewards/_claimExternalRewards, first try to reduce cachedUserRewards, and then increase userRewardDebts
