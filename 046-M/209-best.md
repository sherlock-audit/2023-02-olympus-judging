Udsen

medium

# USER PASSED IN `amount_` IS CONSIDERED FOR INTERNAL CALCULATIONS WITHOUT ACCOUNTING FOR TRANSFER FEES

## Summary

In the `deposit()` function of the `SingleSidedLiquidityVault` contract, the calculation to derive `pairTokenUsed` is as follows: `uint256 pairTokenUsed = amount_ - unusedPairToken`. For the calculation the `amount_` (which is the pairToken amount parameter passed in by the `msg.sender`) is used. But this `_amount` does not take into consideration the transfer fee, which exist in some ERC20 token contracts.

## Vulnerability Detail

If the `pairToken` which is used charges transfer fee during transfer transaction then the actually deposited `pairToken` amount can be less than the `_amount` used in the calculation. 

## Impact

Since the `uint256 pairTokenUsed = amount_ - unusedPairToken` calculation is used to calculate the `pairTokenUsed` amount, and the `amount_` will be more than the actually transfered amount, if transfer fee is charged on the amount, the calculation of the `pairTokenUsed` will retrun a higher value than the actually used pair token value by the protocol. 

Since pairToken is added to the `pairTokenDeposits` mapping and stored on chain to track the user deposit amounts to the contract, users will gain an unfair advantage over the contract, as the stored value as deposits on chain is more than the actually transferred deposit amount by the user. 

## Code Snippet

```solidity
        pairToken.safeTransferFrom(msg.sender, address(this), amount_);
        _borrow(ohmToBorrow);

        uint256 lpReceived = _deposit(ohmToBorrow, amount_, slippageParam_);

        // Calculate amount of pair tokens and OHM unused in deposit
        uint256 unusedPairToken = pairToken.balanceOf(address(this)) - pairTokenBalanceBefore;
        uint256 unusedOhm = ohm.balanceOf(address(this)) - ohmBalanceBefore;

        // Return unused pair tokens to user
        if (unusedPairToken > 0) pairToken.safeTransfer(msg.sender, unusedPairToken);

        // Burn unused OHM
        if (unusedOhm > 0) _repay(unusedOhm);

        uint256 pairTokenUsed = amount_ - unusedPairToken;
```

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L216-L231

## Tool used

VS Code and Manual Review

## Recommendation

It is recommended to locally store the actually transferred amount `pairToken.balanceOf(address(this))` before the `_deposit()` function is called, and use that value to calculate the `pairTokenUsed` and then save it in the `pairTokenDeposits` mapping.

The suggested solution is depicted in the following code snippet.

```solidity
        pairToken.safeTransferFrom(msg.sender, address(this), amount_);
        _borrow(ohmToBorrow);

        uint256 transferedPairToken = pairToken.balanceOf(address(this));

        uint256 lpReceived = _deposit(ohmToBorrow, amount_, slippageParam_);

        // Calculate amount of pair tokens and OHM unused in deposit
        uint256 unusedPairToken = pairToken.balanceOf(address(this)) - pairTokenBalanceBefore;
        uint256 unusedOhm = ohm.balanceOf(address(this)) - ohmBalanceBefore;

        // Return unused pair tokens to user
        if (unusedPairToken > 0) pairToken.safeTransfer(msg.sender, unusedPairToken);

        // Burn unused OHM
        if (unusedOhm > 0) _repay(unusedOhm);

        uint256 pairTokenUsed = transferedPairToken - unusedPairToken;
```

so the final calculation for the `pairTokenUsed` will be as follows: `uint256 pairTokenUsed = transferedPairToken - unusedPairToken`