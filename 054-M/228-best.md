gogo

medium

# No upper bound for `FEE` in SingleSidedLiquidityVault.sol allows `liquidityvault_admin` to set fee to 100% and steal users rewards

## Summary

FEE basis points can be set to 100% (`10e3`) by the `liquidityvault_admin`.

## Vulnerability Detail

Although there is a check for not setting the `FEE` variable to more than `10e3` (more than 100%) it's still an issue that the admin could set the `FEE` to 100% or any percent that will take a huge portion from users rewards. This should be avoided by checking for a maximum value set during deployment e.g. 750 basis points (7.5%).

## Impact

The account with `liquidityvault_admin` role will receive all rewards from users that want to claim their rewards on withdrawal.

## Code Snippet

```solidity
    function setFee(uint256 fee_) external onlyRole("liquidityvault_admin") {
        if (fee_ > PRECISION) revert LiquidityVault_InvalidParams();
        FEE = fee_;
    }
```

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L798-L804

## Tool used

Manual Review

## Recommendation

Add a variable such as `MAX_FEE_BPS` to prevent admin from setting fee too high.
