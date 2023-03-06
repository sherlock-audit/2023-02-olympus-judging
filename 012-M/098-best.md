cccz

medium

# If addExternalRewardToken adds two of the same reward tokens, the user will not be able to claim the reward due to insufficient balance.

## Summary
If addExternalRewardToken adds two of the same reward tokens, the user will not be able to claim the reward due to insufficient balance
## Vulnerability Detail
addExternalRewardToken does not check for token duplication.
```solidity
    function addExternalRewardToken(address token_) external onlyRole("liquidityvault_admin") {
        ExternalRewardToken memory newRewardToken = ExternalRewardToken({
            token: token_,
            decimalsAdjustment: 10**ERC20(token_).decimals(),
            accumulatedRewardsPerShare: 0,
            lastBalance: 0
        });

        externalRewardTokens.push(newRewardToken);
    }
```
_accumulateExternalRewards will use the contract's balance to calculate rewards.
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
```
If addExternalRewardToken adds two of the same reward tokens, then both reward tokens will use the contract's balance as a reward, meaning that the contract will need to pay twice the balance to satisfy the reward. This will result in the user not being able to claim rewards due to insufficient balance.
The POC is as follows
```solidity
    function test_POC() public {
        liquidityVault.addExternalRewardToken(address(externalReward));
        // Setup
        _withdrawTimeSetUp();
        vm.prank(alice);
        liquidityVault.withdraw(1e18, minTokenAmounts_, true);
    }
```
The output
```sh
[FAIL. Reason: TRANSFER_FAILED] test_POC() (gas: 1119229)
```
## Impact
This will result in the user not being able to claim rewards due to insufficient balance
## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L708-L717
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L192-L216
## Tool used

Manual Review

## Recommendation
Performing Token Duplication Checks in addExternalRewardToken
