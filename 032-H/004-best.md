Bauer

high

# deposit()  and withdraw() will revert when Balancer pool is paused.

## Summary

The vault are not able to  deposit() and withdraw().


## Vulnerability Detail
```solidity
 function _deposit(
        uint256 ohmAmount_,
        uint256 pairAmount_,
        uint256 slippageParam_
    ) internal override returns (uint256) {
        // Cast pool address from abstract to Balancer Base Pool
        IBasePool pool = IBasePool(liquidityPool);

        // OHM-wstETH BPT before
        uint256 bptBefore = pool.balanceOf(address(this));

        // Build join pool request
        address[] memory assets = new address[](2);
        assets[0] = address(ohm);
        assets[1] = address(pairToken);

        uint256[] memory maxAmountsIn = new uint256[](2);
        maxAmountsIn[0] = ohmAmount_;
        maxAmountsIn[1] = pairAmount_;

        JoinPoolRequest memory joinPoolRequest = JoinPoolRequest({
            assets: assets,
            maxAmountsIn: maxAmountsIn,
            userData: abi.encode(1, maxAmountsIn, slippageParam_),
            fromInternalBalance: false
        });

        // Join Balancer pool
        ohm.approve(address(vault), ohmAmount_);
        pairToken.approve(address(vault), pairAmount_);
        vault.joinPool(pool.getPoolId(), address(this), address(this), joinPoolRequest);

        // OHM-PAIR BPT after
        uint256 lpAmountOut = pool.balanceOf(address(this)) - bptBefore;

        // Stake into Aura
        pool.approve(address(auraPool.booster), lpAmountOut);
        auraPool.booster.deposit(auraPool.pid, lpAmountOut, true);

        return lpAmountOut;
    }
```
Balancer pool is designed such that, it will allow joinPool or exitPool() only when it is not paused. Please refer following link. 

https://github.com/Cron-Finance/v1-periphery/blob/bee861678ef7c41193b6eedff5732c2bf6929351/src/balancer-core-v2/vault/PoolBalances.sol#L43-L54

When Balancer pool  is paused, it can not deposit() and withdraw().

## Impact
The vault are not able to  deposit() and withdraw().

## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L187-L244
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L252-L285

## Tool used

Manual Review

## Recommendation
Balancer implemented the pause mechanism to tackle the certain emergence situations. It is suggetsed to follow the same mechanism for olympus as well. If Balancer pool joinpool() and exitpool() are paused, pause the olympus 's deposit() and withdraw() too.