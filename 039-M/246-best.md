ak1

medium

# SingleSidedLiquidityVault.sol : PRECISION is recommened as 1000000 instead of 1000

## Summary

SingleSidedLiquidityVault.sol contract uses the PRECISION as 1000.

since the ERC tokens mentioned in the document has different decimal values, precision based calculation could affect the AURA.

`OHM, wstETH, LDO, BAL, ---> 18 decimal`

`AURA -> 6 decimal.`

## Vulnerability Detail

Refer the summary section.

## Impact

AURA token could suffer due to precision related calculation.

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L111

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L417-L418

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L626

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L639

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L793-L804

## Tool used

Manual Review

## Recommendation

We have see the many of the protocols are using the precision as 1000000.
We suggest to follow the same here to avoid any of the calculation approximation.
