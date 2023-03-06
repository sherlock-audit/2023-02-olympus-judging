peanuts

medium

# auraPool.rewardsPool.withdrawAndUnwrap does not check boolean value

## Summary

 auraPool.rewardsPool.withdrawAndUnwrap does not check boolean value.

## Vulnerability Detail

withdrawAndUnwrap() returns a boolean value whenever the function is called.

https://etherscan.io/address/0x00a7ba8ae7bca0b10a32ea1f8e2a1da980c6cad2

In this protocol, the function does not check the boolean value.

```solidity
        auraPool.rewardsPool.withdrawAndUnwrap(lpAmount_, false);
```

## Impact

Function may silently fail.

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L175

## Tool used

Manual Review

## Recommendation

Check the boolean value like this:

```solidity
        bool success = auraPool.rewardsPool.withdrawAndUnwrap(lpAmount_, false);
        require(success, 'withdraw failed');
```

An example: 

https://github.com/sherlock-audit/2022-12-notional/blob/55b3b0a451331e198fcb28714a0dbd6dabda38c1/contracts/vaults/balancer/internal/pool/TwoTokenPoolUtils.sol#L284-L285