GimelSec

medium

# `abstracts/SingleSidedLiquidityVault.sol` doesn't handle fee-on-transfer pairToken

## Summary

`abstracts/SingleSidedLiquidityVault.sol` doesn't handle fee-on-transfer pairToken.

## Vulnerability Detail

In the abstract contract `SingleSidedLiquidityVault.sol`, it directly uses the `amount_` parameter, values the collateral, and records `amount` directly into `pairTokenUsed`.

```solidity
        // Gather tokens for deposit
        pairToken.safeTransferFrom(msg.sender, address(this), amount_);
        _borrow(ohmToBorrow);

        uint256 lpReceived = _deposit(ohmToBorrow, amount_, slippageParam_);

        uint256 pairTokenUsed = amount_ - unusedPairToken;
```

Assume that the pairToken is a fee-on-transfer token with 10% fees, and the current balance of pairToken is 100 (The protocol may have some remaining `pairToken`, a scenario is that the external reward token is the same as `pairToken` with 100 tokens).
If Alice deposits 100 tokens, the protocol will receive 90 tokens, `_deposit()` will not be underflow due to the remaining pairToken.
Then if `_deposit()` only used 80%, `unusedPairToken` will be `(100 + 90 - 100*0.8) - 100 = 10`, which should be `(100 + 90 - 90*0.8) - 100 = 18`. The `pairTokenUsed` will get `100 - 10 = 90` which should be `90 - 18 = 72`, Alice will get a higher value of `pairTokenDeposits`.
Furthermore, the `ohmToBorrow` will also be wrong due to a higher value from `_valueCollateral(amount_)`.

## Impact

Users will get higher values of `pairTokenUsed` and `lpReceived`, the protocol will lose tokens to compensate users for the execution.

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L215-L219
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L200-L201
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L231

## Tool used

Manual Review

## Recommendation

Calculate an actual amount before `_deposit()`:

```diff
        ...

-       // Calculate amount of OHM to borrow
-       uint256 ohmToBorrow = _valueCollateral(amount_);

        // Cache pair token and OHM balance before deposit
        uint256 pairTokenBalanceBefore = pairToken.balanceOf(address(this));
        uint256 ohmBalanceBefore = ohm.balanceOf(address(this));

        // The pool being imbalanced is less of a concern here on deposit than on withdrawal,
        // but in the event the frontend miscalculates the expected LP amount to receive, we want
        // to reduce the risk of entering a manipulated pool at a bad price
        if (!_isPoolSafe()) revert LiquidityVault_PoolImbalanced();
        if (!_canDeposit(ohmToBorrow)) revert LiquidityVault_LimitViolation();

        _depositUpdateRewardState();

        // Gather tokens for deposit
+       uint256 amountBefore = pairToken.balanceOf(address(this));
        pairToken.safeTransferFrom(msg.sender, address(this), amount_);
+       uint256 received = pairToken.balanceOf(address(this)) - amountBefore;

+       // Calculate amount of OHM to borrow
+       uint256 ohmToBorrow = _valueCollateral(received);

+       _borrow(ohmToBorrow);

+       uint256 lpReceived = _deposit(ohmToBorrow, received, slippageParam_);

        ...

+       uint256 pairTokenUsed = received - unusedPairToken;

        ...
    }
```