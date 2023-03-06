ABA

medium

# `getMaxDeposit` returns less then it should; possibly resulting in fewer tokens being bought

## Summary

Function `getMaxDeposit` in `SingleSidedLiquidityVault.sol` returns less then it should, possibly resulting in fewer tokens being bought by simply offering erroneous values to potential buyers, making them buy less then they could have bought.

## Vulnerability Detail

The `getMaxDeposit` calculates the max deposit by:
- retrieving the current pooled OHM shares (`currentPoolOhmShare`)
- calculating the emitted (if there are any) OHM by deducting the minted OHM and the currently pooled OHM

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L318-L323

- deducts from the max allowed minted OHM emitted and minted, among others

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L324

- gets the price token pair <-> OHM equirevalent exchange

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L326-L327

- adjusts for pair token decimals

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L328

- returns the max amount to deposit

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L329

What the function fails to take into consideration is already removed OHM (the `ohmRemoved`).

A correct way to get the emissions is shown via the `getOhmEmissions` function

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L392-L396

Because of the way that the output of `getMaxDeposit` is calculated, the incorrect implementation returns a less then real value.

## Impact

Although a rare case, any user that wishes to deposit based on the maximum available amount (e.g. 10% of max possible), and gets this value via `getMaxDeposit` will receive a lower then intended value. This basically deprives, indirectly, the protocol of extra liquidity.

## Code Snippet

## Tool used

Manual Review

## Recommendation

Make `getOhmEmissions` public and reuse it in the `getMaxDeposit` function. Example implementation:
```Solidity
    function getMaxDeposit() public view returns (uint256) {
        (uint256 emitted, uint256 removed) = getOhmEmissions();
        uint256 maxOhmAmount = LIMIT + ohmRemoved - ohmMinted - emitted;

        // Convert max OHM mintable amount to pair token amount
        uint256 ohmPerPairToken = _valueCollateral(1e18); // OHM per 1 pairToken
        uint256 pairTokenDecimalAdjustment = 10**pairToken.decimals();
        return (maxOhmAmount * pairTokenDecimalAdjustment) / ohmPerPairToken;
    }
```