peanuts

medium

# auraPool.booster.deposit boolean return value not handled in WstethLiquidityVault.sol

## Summary

auraPool.booster.deposit boolean return value not handled in WstethLiquidityVault.sol

## Vulnerability Detail

The function _deposit() in WstethLiquidityVault.sol is called from deposit#SingleSidedLiquidityVault.sol, and its main aim is to get OHM-PAIR BPT LP tokens and stake the tokens into the aura pool.

```solidity
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
```

The deposit function in the Aura implementation returns a boolean to acknowledge that the deposit is successful.

https://etherscan.io/address/0x7818A1DA7BD1E64c199029E86Ba244a9798eEE10#code#F34#L1

```solidity
function deposit(uint256 _pid, uint256 _amount, bool _stake) public returns(bool){
```

## Impact

If the boolean value is not handled, the transaction may fail silently.

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L123-L140

## Tool used

Manual Review

## Recommendation

Recommend checking for success return value 

```solidity
-        auraPool.booster.deposit(auraPool.pid, lpAmountOut, true);
+        bool success = auraPool.booster.deposit(auraPool.pid, lpAmountOut, true);
+        require(success, 'stake failed')
```

like how this protocol does it:

https://github.com/sherlock-audit/2022-12-notional/blob/55b3b0a451331e198fcb28714a0dbd6dabda38c1/contracts/vaults/balancer/internal/pool/TwoTokenPoolUtils.sol#L272-L273
