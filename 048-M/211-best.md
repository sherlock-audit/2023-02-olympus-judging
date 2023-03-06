Bahurum

medium

# Rewards can get stuck and can be unavailable for some time

## Summary
When an user withdraws with claim but the SSLV does not have enough rewards, the rewarding logic is supposed to cache the remainder that could not be sent to the claimer for later. In the implementation, the claims revert if they exceed the rewards held by the SSLV so the caching mechanism in `_withdrawUpdateRewardState()` doesn't work. This leads to rewards unavailable for some users when the vault is short on rewards.

## Vulnerability Detail
In `_claimInternalRewards()` the full rewards (minus fees) are sent to the claimer:
```solidity 
uint256 reward = internalRewardsForToken(id_, msg.sender);
...
if (reward > 0) ERC20(rewardToken).safeTransfer(msg.sender, reward - fee);
```
This will revert in case the SSLV does not have enough balance of the reward token.

The same issue is present for external rewards.

Scenario:
1. Alice deposits 1 wstETH
2. 1 moth passes, she is eligible to 100 LDO of internal rewards, but LDO yield went down since and only 99 LDO are in the vault
3. Alice wants to withdraw and claim the rewards to invest the rewards elsewhere: she should be able to claim the 99 LDO and 1 LDO should be accounted as cached in the contract, but instead the tx reverts and the rewards are stuck
4. After some days governance multisig approves a transfer of LDO to the SSLV to unstuck the rewards
5. Alice can finally claim 100 LDO and invest them elsewhere, but she lost some days of returns on the investment.

# PoC
Add this test to `WstethLiquidityVaultMock.t.sol`:

```solidity
    function testExhaustedRewards() public {
        // Setup
        _withdrawTimeSetUp();
        // let the rewards accumulate to more than balance of LDO
        (
            address token,
            uint256 decimals,
            uint256 rewardsPerSecond,
            ,
            uint256 accumulatedRewardsPerShare
        ) = liquidityVault.internalRewardTokens(0);
        assert(token == address(reward));
        uint timeToExhaustBalance = reward.balanceOf(address(liquidityVault))/rewardsPerSecond;
        vm.warp(block.timestamp + timeToExhaustBalance);
        ohmEthPriceFeed.setTimestamp(block.timestamp);
        ethUsdPriceFeed.setTimestamp(block.timestamp);
        stethUsdPriceFeed.setTimestamp(block.timestamp); 
        console2.log("time elapsed:" , timeToExhaustBalance);
        console2.log("alice balance of reward before:" , reward.balanceOf(alice));
        // try to get rewards: more than available in vault
        uint aliceLpTokens = liquidityVault.lpPositions(alice);
        vm.expectRevert();
        vm.prank(alice);
        liquidityVault.withdraw(1, minTokenAmounts_, true);
        console2.log("alice balance of reward after:" , reward.balanceOf(alice));
    }
```
## Impact
If the rate of rewards tokens accumlated in the pool becomes slower than `rewardsPerSecond_` (for example the yield becomes smaller over time), all the rewards can be claimed and when no more rewards are available, the protocol must intervene and send rewards token manually to the SSLV. If the governance is a multisig, it can take days before some users can claim rewards again.

## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L623-L648

## Tool used

Manual Review

## Recommendation
In `_claimInternalRewards()`, cap the rewards to be sent to the balance of reward token of the SSLV.
```solidity
    ...
    uint256 reward = internalRewardsForToken(id_, msg.sender);
+   uint256 rewardBalance = ERC20(rewardToken).balanceOf(address(this));
+   reward = reward > rewardBalance ? rewardBalance : reward
    uint256 fee = (reward * FEE) / PRECISION;
    ...
```
Same should be done for external rewards.