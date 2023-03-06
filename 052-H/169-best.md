tsvetanovv

high

# `bptTotalSupply` will return an incorrect value

## Summary
In [WstethLiquidityVault.sol](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L274) we have `_getPoolOhmShare()` and `getUserWstethShare()` functions. These functions will return an incorrect value of `bptTotalSupply` because they use `totalSupply` instead of `virtualSupply`.
## Vulnerability Detail
Function `_getPoolOhmShare()` calculates the vault's claim on OHM in the Balancer pool. The `totalBPTSupply` will be excessively inflated as `totalSupply` was used instead of `virtualSupply`. This might cause a boosted balancer leverage vault not to be emergency settled in a timely manner and holds too large of a share of the liquidity within the pool, thus having problems exiting its position.
This issue is specific to the Balancer pool.
https://docs.balancer.fi/reference/lp-tokens/underlying.html
## Impact
`bptTotalSupply` will return the wrong value.
## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L274
```solidity
function _getPoolOhmShare() internal view override returns (uint256) {

// Cast pool address from abstract to Balancer Base Pool

IBasePool pool = IBasePool(liquidityPool);

(, uint256[] memory balances_, ) = vault.getPoolTokens(pool.getPoolId());

uint256 bptTotalSupply = pool.totalSupply();

if (totalLP == 0) return 0;

else return (balances_[0] * totalLP) / bptTotalSupply;

}
```
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L326
```solidity
function getUserWstethShare(address user_) internal view returns (uint256) {

// Cast pool address from abstract to Balancer Base pool

IBasePool pool = IBasePool(liquidityPool);

// Get user's LP balance

uint256 userLpBalance = lpPositions[user_];

(, uint256[] memory balances_, ) = vault.getPoolTokens(pool.getPoolId());

uint256 bptTotalSupply = pool.totalSupply();

return (balances_[1] * userLpBalance) / bptTotalSupply;

}
```
## Tool used

Manual Review

## Recommendation
Update the function to compute the `totalBPTSupply` from the virtual supply.
Use `virtualSupply` instead `totalSupply`.