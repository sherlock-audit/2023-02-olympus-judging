carrot

high

# Incorrect handling of rewards in `withdraw` function

## Summary
The function `withdraw` in `SingleSidedLiquidityVault.sol` takes in an argument `bool claim` which can be set to true to withdraw rewards. However, the maths in this function is incorrect, allowing users to withdraw double rewards.
## Vulnerability Detail
According to the specification, calling the function `withdraw(lp,mintok,true)` should withdraw all pending rewards since `true` is being sent to `bool claim_`. This is not true, and can be observed using a simple test case, which shows that even after calling the function with `true` claim, the contract stores pending rewards for the user, allowing a second reward claim. Observable using this test snippet
```solidity
function testATTACKreward() public {
        _withdrawTimeSetUp();

        // 1. Withdraw and claim
        vm.startPrank(alice);
        assertEq(reward.balanceOf(alice), 0);
        liquidityVault.withdraw(1e18, minTokenAmounts_, true);
        emit log_named_uint("reward after withdraw", reward.balanceOf(alice));

        // 2. Pending rewards claim
        emit log_named_uint(
            "Pending rewards",
            liquidityVault.internalRewardsForToken(0, alice)
        );
        liquidityVault.claimRewards();
        // Verify end state
        emit log_named_uint(
            "reward after withdraw + claim",
            reward.balanceOf(alice)
        );
        vm.stopPrank();
    }
```
Which when run gives the following output
![image](https://user-images.githubusercontent.com/108595272/220715028-869d949a-adaa-4767-b3d8-384ee0c07d19.png)

This shows that the rewards are paid out, but also cached, allowing a second payout.

The reason for this can be found in the contract `SingleSidedLiquidityVault`
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L574-L595

This function is called during execution of the `withdraw` function and handles the calculation of the reward state. The two contradicting pieces of code are in the function call to `_claimInternalRewards`, and the if block following it.
1. In the function `_claimInternalRewards`, a reward value is calculated and sent to the user, and the debt is updated
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L625-L631
The reward amount calculated is implemented in `internalRewardsForToken` function, and is essentially equal to `AMOUNT = lp[user] * reward.accumulatedRewardsPerShare` if the current user debt and cached debt is 0 (for simpler calculations). The reward debt of the user is updated to this same value after this step, and the payout is done.
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L368-L369
2. The above-mentioned user debt update is incorrectly **undone** in the following if block. 
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L581
The value `rewardDebtDiff` is calculated to be the same value as `AMOUNT` calculated in the last step. This is because the accumulation steps were done before either the steps 1 or 2 mentioned here. Thus at this point, `rewardDebtDiff = userRewardDebts[msg.sender][rewardToken.token]`, since the debts were updated while claiming. This hits the `else` block, which wipes the debt, allowing the user to claim rewards again
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L588-L590 

The same issue also exists with the calculation of external rewards

## Impact
Wrong rewards calculations for users during `withdraw`
## Code Snippet
POC:
```solidity
function testATTACKreward() public {
        _withdrawTimeSetUp();

        // 1. Withdraw and claim
        vm.startPrank(alice);
        assertEq(reward.balanceOf(alice), 0);
        liquidityVault.withdraw(1e18, minTokenAmounts_, true);
        emit log_named_uint("reward after withdraw", reward.balanceOf(alice));

        // 2. Pending rewards claim
        emit log_named_uint(
            "Pending rewards",
            liquidityVault.internalRewardsForToken(0, alice)
        );
        liquidityVault.claimRewards();
        // Verify end state
        emit log_named_uint(
            "reward after withdraw + claim",
            reward.balanceOf(alice)
        );
        vm.stopPrank();
    }
```
## Tool used

Foundry tests
## Recommendation
Reward Debts incorrectly being reduced after claims. Reducing reward debts allow for more rewards to be claimed. Since there are no token transfers being done, there is no need to update `userRewardDebts` in the `else` block. Reward Debts should only be updated after transfers, and NEVER reduced. 