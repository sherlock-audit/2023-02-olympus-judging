0x52

medium

# rescueToken doesn't update rewardToken.lastBalance for external reward tokens

## Summary

SingleSidedLiquidityVault allows the admin tokens from the vault contract. This can only be done once the vault has been deactivated but there is nothing stopping the contract from being reactivated after a token has been rescued. If an external reward token is rescued then the token accounting will be permanently broken after when/if the vault is re-enabled.

## Vulnerability Detail

See summary.

## Impact

External reward tokens are broken after being rescued

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L774-L780

## Tool used

ChatGPT

## Recommendation

If the token being rescued is an external reward token then rescueToken should update rewardToken.lastBalance