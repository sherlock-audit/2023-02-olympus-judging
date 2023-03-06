Bauer

high

# Users can get double rewards

## Summary
```rewardDebtDiff``` is always greater than  ```userRewardDebts[msg.sender][rewardToken] resulting in double rewards for users.
## Vulnerability Detail
```solidity
function withdraw(
        uint256 lpAmount_,
        uint256[] calldata minTokenAmounts_,
        bool claim_
    ) external onlyWhileActive nonReentrant returns (uint256) {
        // Liquidity vaults should always be built around a two token pool so we can assume
        // the array will always have two elements
        if (lpAmount_ == 0 || minTokenAmounts_[0] == 0 || minTokenAmounts_[1] == 0)
            revert LiquidityVault_InvalidParams();
        if (!_isPoolSafe()) revert LiquidityVault_PoolImbalanced();

        _withdrawUpdateRewardState(lpAmount_, claim_);

        totalLP -= lpAmount_;
        lpPositions[msg.sender] -= lpAmount_;

        // Withdraw OHM and pairToken from LP
        (uint256 ohmReceived, uint256 pairTokenReceived) = _withdraw(lpAmount_, minTokenAmounts_);

        // Reduce deposit values
        uint256 userDeposit = pairTokenDeposits[msg.sender];
        pairTokenDeposits[msg.sender] -= pairTokenReceived > userDeposit
            ? userDeposit
            : pairTokenReceived;
        ohmMinted -= ohmReceived > ohmMinted ? ohmMinted : ohmReceived;
        ohmRemoved += ohmReceived > ohmMinted ? ohmReceived - ohmMinted : 0;

        // Return assets
        _repay(ohmReceived);
        pairToken.safeTransfer(msg.sender, pairTokenReceived);

        emit Withdraw(msg.sender, pairTokenReceived, ohmReceived);
        return pairTokenReceived;
    }

```
The protocol allows user to withdraw pair tokens and claim rewards if the ```bool claim_``` in the function ```withdraw()``` parameter is true.There is an internal call to the function ```_withdrawUpdateRewardState()```.In this function, the protocol first calculates internal and external rewards and update reward state. Next, call to function ```_claimInternalRewards()``` to distribute reward to user. The function ```internalRewardsForToken()``` is used to calculating user rewards. The protocol will transfer all the reward during this period to the user and   update  the value to the variable ```userRewardDebts[msg.sender][rewardToken] += reward``` (18 decimals). Moreover, the protocol  will update reward debts so as to not understate the amount of rewards owed to the user,and push any unclaimed rewards to the user's reward debt so that they can be claimed later.The parameter ```rewardDebtDiff ``` involved in the calculation are calculated according to this formula  ```rewardDebtDiff = lpAmount_ * rewardToken.accumulatedRewardsPerShare```.  the lpAmount is 18 decimals, the rewardToken.accumulatedRewardsPerShare is also 18 decimals, so the rewardDebtDiff is 36 decimals. This condition is always true. ``` if (rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]```, there is a lot of rewards will be pushed to  user's reward debt ```cachedUserRewards[msg.sender][rewardToken.token] +=
                    rewardDebtDiff -
                    userRewardDebts[msg.sender][rewardToken.token]```. User can call function ```claimRewards()``` to get rewards again.

1.Alice calls function ```withdraw()``` to withdraw all her pair tokens and claim rewards and specifed the ```bool claim_``` in the function  parameter true
2. The protocol distribute rewards to user.
userRewardDebts[msg.sender][rewardToken] += reward (18 decimals)

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

3.The protocol update reward debts so as to not understate the amount of rewards owed to the user, and push any unclaimed rewards to the user's reward debt so that they can be claimed later.
 rewardDebtDiff = lpAmount_ * rewardToken.accumulatedRewardsPerShare  (36 decimals)
rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token] = true
cachedUserRewards[msg.sender][rewardToken.token] += rewardDebtDiff - userRewardDebts[msg.sender][rewardToken.token] (36 decimals).

4.Alice calls function ```claimRewards()```  to get the cached reward

```solidity
for (uint256 i; i < numInternalRewardTokens; ) {
            _updateInternalRewardState(i, accumulatedInternalRewards[i]);

            if (claim_) _claimInternalRewards(i);

            InternalRewardToken memory rewardToken = internalRewardTokens[i];
            uint256 rewardDebtDiff = lpAmount_ * rewardToken.accumulatedRewardsPerShare;

            if (rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]) {
                userRewardDebts[msg.sender][rewardToken.token] = 0;
                cachedUserRewards[msg.sender][rewardToken.token] +=
                    rewardDebtDiff -
                    userRewardDebts[msg.sender][rewardToken.token];
            } else {
                userRewardDebts[msg.sender][rewardToken.token] -= rewardDebtDiff;
            }

            unchecked {
                ++i;
            }
        }

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


## Impact
User can get double rewards and the protocol will loss assets.

## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L252-L285
## Tool used

Manual Review

## Recommendation
rewardDebtDiff divided by 1e

```solidity
  // any unclaimed rewards to the user's reward debt so that they can be claimed later
            InternalRewardToken memory rewardToken = internalRewardTokens[i];
            uint256 rewardDebtDiff = lpAmount_ * rewardToken.accumulatedRewardsPerShare;

            if (rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]*1e18) {
                userRewardDebts[msg.sender][rewardToken.token] = 0;
                cachedUserRewards[msg.sender][rewardToken.token] +=
                    rewardDebtDiff -
                    userRewardDebts[msg.sender][rewardToken.token];
            } else {
                userRewardDebts[msg.sender][rewardToken.token] -= rewardDebtDiff;
            }
```
