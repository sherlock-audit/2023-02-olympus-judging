0x52

high

# Removed reward tokens will no longer be claimable and will cause loss of funds to users who haven't claimed

## Summary

When a reward token is removed, it's entire reward structs is deleted from the reward token array. The results is that after it has been removed it is impossible to claim. User's who haven't claimed will permanently lose all their unclaimed rewards.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L694-L703

When a reward token is removed the entire reward token struct is deleted from the array

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L288-L310

When claiming rewards it cycles through the current reward token array and claims each token. As a result of this, after a reward token has been removed it becomes impossible to claim. Any unclaimed balance that a user had will be permanently lost.

Submitting this as high because the way that internal tokens are accrued (see "Internal reward tokens can and likely will over commit rewards") will force this issue and therefore loss of funds to users to happen.

## Impact

Users will lose all unclaimed rewards when a reward token is removed

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L694-L703

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L723-L732

## Tool used

ChatGPT

## Recommendation

When a reward token is removed it should be moved into a "claim only" mode. In this state rewards will no longer accrue but all outstanding balances will still be claimable.