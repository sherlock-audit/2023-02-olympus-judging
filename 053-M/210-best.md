Bahurum

medium

# Vault can experience long downtime periods

## Summary
The chainlink price could stay up to 24 hours (heartbeat period) outside the boundaries defined by `THRESHOLD` but within the chainlink deviation threshold. Deposits and withdrawals will not be possible during this period of time.

## Vulnerability Detail
The `_isPoolSafe()` function checks if the balancer pool spot price is within the boundaries defined by `THRESHOLD` respect to the last fetched chainlink price. 

Since in `_valueCollateral()` the `updateThreshold` should be 24 hours (as in the tests), then the OHM derived oracle price could stay at up to 2% from the on-chain trusted price. The value is 2% because in [WstethLiquidityVault.sol#L223](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L223):
```solidity
return (amount_ * stethPerWsteth * stethUsd * decimalAdjustment) / (ohmEth * ethUsd * 1e18);
```
`stethPerWsteth` is mostly stable and changes in `stethUsd` and `ethUsd` will cancel out, so the return value changes will be close to changes in `ohmEth`, so up to 2% from the on-chain trusted price.

If `THRESHOLD` < 2%, say 1% as in the tests, then the Chainlink price can deviate by more than 1% from the pool spot price and less than 2% from the on-chain trusted price fro up to 24 h. During this period withdrawals and deposits will revert.

## Impact
Withdrawals and deposits can be often unavailable for several hours.
## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L411-L421

## Tool used

Manual Review

## Recommendation
`THRESHOLD` is not fixed and can be changed by the admin, meaning that it can take different values over time.Only a tight range of values around 2% should be allowed to avoid the scenario above.
