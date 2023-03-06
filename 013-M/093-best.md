hansfriese

medium

# `SingleSidedLiquidityVault.getMaxDeposit()` uses the wrong decimals.



## Summary
In `getMaxDeposit()`, it uses 18 instead of `pairToken` decimals while converting OHM to `pairToken`.

## Vulnerability Detail
`getMaxDeposit()` returns max amount of pair token that can be deposited.

```solidity
    function getMaxDeposit() public view returns (uint256) {
        uint256 currentPoolOhmShare = _getPoolOhmShare();
        uint256 emitted;

        // Calculate max OHM mintable amount
        if (ohmMinted > currentPoolOhmShare) emitted = ohmMinted - currentPoolOhmShare;amount of OHM, compare amount and share
        uint256 maxOhmAmount = LIMIT + ohmRemoved - ohmMinted - emitted;

        // Convert max OHM mintable amount to pair token amount
        uint256 ohmPerPairToken = _valueCollateral(1e18); // OHM per 1 pairToken //@audit use pairToken decimal instead of 18
        uint256 pairTokenDecimalAdjustment = 10**pairToken.decimals();
        return (maxOhmAmount * pairTokenDecimalAdjustment) / ohmPerPairToken;
    }
```

But when it calculates `ohmPerPairToken` using `_valueCollateral()`, it uses `1e18` for 1 `pairToken` and it will be wrong if `pairToken` decimals != 18.

## Impact
`getMaxDeposit()` will return the wrong result.

## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L327

## Tool used
Manual Review

## Recommendation
It should be modified like below.

```solidity
    // Convert max OHM mintable amount to pair token amount
    uint256 pairTokenDecimalAdjustment = 10**pairToken.decimals();
    uint256 ohmPerPairToken = _valueCollateral(pairTokenDecimalAdjustment); // OHM per 1 pairToken
    return (maxOhmAmount * pairTokenDecimalAdjustment) / ohmPerPairToken;
```