Ruhum

medium

# Chainlink has no stETH/USD feed on mainnnet

## Summary
The WstethLiquidityVault depends on a stETH/USD oracle. But, Chainlink doesn't have that feed on Mainnet. It only exists on Optimism & Arbitrum: https://data.chain.link/optimism/mainnet/crypto-usd/steth-usd

## Vulnerability Detail
In `WstethLiquidityVault._valueCollateral()` the contract needs the stETH/USD price to figure out how many OHM tokens the given amount of wstETH is worth. But, since that feed doesn't exist, the function won't return the correct value.

## Impact
The contract won't be deployable because of a dependency on a non-existent Chainlink feed

## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L239

```sol
    /// @notice                 Calculates the OHM equivalent quantity for the wstETH deposit
    /// @param amount_          Amount of wstETH to calculate OHM equivalent for
    /// @return uint256         OHM equivalent quantity
    function _valueCollateral(uint256 amount_) public view override returns (uint256) {
        uint256 stethPerWsteth = IWsteth(address(pairToken)).stEthPerToken();

        // This is returned in 18 decimals and represents ETH per OHM
        uint256 ohmEth = _validatePrice(
            address(ohmEthPriceFeed.feed),
            uint256(ohmEthPriceFeed.updateThreshold)
        );

        // This is returned in 8 decimals and represents USD per ETH
        uint256 ethUsd = _validatePrice(
            address(ethUsdPriceFeed.feed),
            uint256(ethUsdPriceFeed.updateThreshold)
        );

        // This is returned in 8 decimals and represents USD per stETH
        uint256 stethUsd = _validatePrice(
            address(stethUsdPriceFeed.feed),
            uint256(stethUsdPriceFeed.updateThreshold)
        );

        // Amount is 18 decimals in the case of wstETH and OHM has 9 decimals so to get a result with 9
        // decimals we need to use this decimal adjustment
        uint8 ohmDecimals = 9;
        uint256 decimalAdjustment = 10 **
            (ohmEthPriceFeedDecimals +
                ethUsdPriceFeedDecimals +
                ohmDecimals -
                stethUsdPriceFeedDecimals -
                pairTokenDecimals);

        return (amount_ * stethPerWsteth * stethUsd * decimalAdjustment) / (ohmEth * ethUsd * 1e18);
    }
```

## Tool used

Manual Review

## Recommendation
On mainnet, there's only a stETH/ETH feed. That means you have to convert wstETH -> stETH -> ETH. With the OHM/ETH feed, you can then figure out much OHM the given amount of wstETH is worth.
