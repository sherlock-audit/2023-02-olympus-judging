cducrest-brainbot

high

# Vault may prevent withdraw / deposit due to outdated oracle price

## Summary

Chainlink aggregators have a built in circuit breaker if the price of an asset goes outside of a predetermined price band. The result is that if an asset experiences a huge drop in value (i.e. LUNA crash) the price of the oracle will continue to return the minPrice instead of the actual price of the asset. This is exactly what happened to [Venus on BSC when LUNA imploded](https://rekt.news/venus-blizz-rekt/).

If the underlying pool is open to the market, the price of the pool will reflect the real price of the token pair and will not match the value returned by the oracle.

## Vulnerability Detail

The difference in price in between what is returned by the oracle and the pool will prevent people from depositing but most importantly withdrawing from the vault due to `_isPoolSafe()` failing.

## Impact

When real token pair / ohm price gets beyond the limit of the price oracle, users will not be able to recover their pair tokens out of the vault.

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L411-L421

## Tool used

Manual Review

## Recommendation

Allow emergency withdrawal when the price of the oracles drop / go too high for the user.