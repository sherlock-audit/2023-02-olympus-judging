carrot

medium

# Absence of `emergencyWithdraw` feature

## Summary
In contracts handling multiple rewards tokens such as this, it is always recommended to implement an emergency withdraw feature, which forfeits rewards and withdraws the LPtokens. This feature is missing in this contract.
## Vulnerability Detail
An emergency withdraw feature allows users to withdraw their LP Tokens, forfeiting all their reward tokens. This is important in a number of scenarios:
1. Incorrect accounting of rewards: if the rewards accounting ever breaks, either due to bugs or tampering, an emergency withdraw feature bypasses the reward token calculation feature which might be reverting due to underflow/overflow errors, as is shown in another report submitted. This lets users recover at least their initial investment.
2. High gas costs: The standard withdraw function interacts with the balancer and aura vaults, along with an uncapped multitude of internal and external reward tokens. This can increase gas costs significantly, and thus an alternative to recover the underlying should be provided.
3. Block gas limit: The withdraw mechanism loops over an uncapped number of reward tokens, both external and internal, multiple times (`_withdrawUpdateRewardState`, `_accumulateInternalRewards`, `_accumulateExternalRewards`). 
If the list of reward tokens were to grow large enough that the multiple for loops hit the block gas limit, withdraws would become impossible.

Due to these various reasons, an `emergencyWithdraw` mechanism should be established, which simply cashes out ALL LPtokens, without accumulating the individual rewards.
## Impact
High gas costs, might lead to out of gas errors or reverts during withdraws
## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L574
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L597
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L467
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L198
## Tool used

Manual Review

## Recommendation
Implement an `emergencyWithdraw` function which does the following:
1. Refund ALL LPTokens
Since ALL LPTokens of the user are refunded, there is no need to update `userRewardDebt` values since they will be updated once the user enters the contract with a `deposit` call.