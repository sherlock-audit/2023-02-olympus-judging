0x52

high

# User can drain entire reward balance due to accounting issue in _claimInternalRewards and _claimExternalRewards

## Summary

The `userRewardDebt`s array stores the users debt to 36 dp but in `_claimInternalRewards` and `_claimExternalRewards` the 18 dp reward token amount. The result is that `usersRewardDebts` incorrectly tracks how many rewards have been claimed and would allow an adversary to claim repeatedly and drain the entire reward balance of the contract.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L368-L369

When calculating the total rewards owed a user it subtracts `userRewardDebts` from `lpPositions[user_] * accumulatedRewardsPerShare`. Since `lpPositions[user_]` and `accumulatedRewardsPerShare` are both 18 dp values, this means that `userRewardDebts` should store the debt to 36 dp. 

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L542-L545

In `_depositUpdateRewardDebts` we can see that `userRewardDebts` is in fact stored as a 36 dp value because `lpReceived_` and  `rewardToken.accumulatedRewardsPerShare` are both 18 dp values.

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L623-L634

When claiming tokens, `userRewardDebts` is updated with the raw 18 dp `reward` amount NOT a 36 dp value like it should. The result is that `userRewardDebts` is incremented by a fraction of what it should be. Since it isn't updated correctly, subsequent claims will give the user too many tokens. An malicious user could abuse this to repeatedly call the contract and drain it of all reward tokens.

## Impact

Contract will send to many reward tokens and will be drained

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L623-L634

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L636-L647

## Tool used

ChatGPT

## Recommendation

Scale the `reward` amount by 1e18:

        uint256 fee = (reward * FEE) / PRECISION;

    -   userRewardDebts[msg.sender][rewardToken.token] += reward;
    +   userRewardDebts[msg.sender][rewardToken.token] += reward * 1e18;