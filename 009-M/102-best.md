cccz

medium

# SingleSidedLiquidityVault.withdraw will decreases ohmMinted, which will make the calculation involving ohmMinted incorrect

## Summary
SingleSidedLiquidityVault.withdraw will decreases ohmMinted, which will make the calculation involving ohmMinted incorrect.
## Vulnerability Detail
In SingleSidedLiquidityVault, ohmMinted indicates the number of ohm minted in the contract, and ohmRemoved indicates the number of ohm burned in the contract.
So the contract just needs to increase ohmMinted in deposit() and increase ohmRemoved in withdraw().
But withdraw() decreases ohmMinted, which makes the calculation involving ohmMinted incorrect.
```solidity
        ohmMinted -= ohmReceived > ohmMinted ? ohmMinted : ohmReceived;
        ohmRemoved += ohmReceived > ohmMinted ? ohmReceived - ohmMinted : 0;
```
Consider that a user minted 100 ohm in deposit() and immediately burned 100 ohm in withdraw().

In \_canDeposit, the amount_ is less than LIMIT + 1000 instead of LIMIT 
```solidity
    function _canDeposit(uint256 amount_) internal view virtual returns (bool) {
        if (amount_ + ohmMinted > LIMIT + ohmRemoved) revert LiquidityVault_LimitViolation();
        return true;
    }

```
getOhmEmissions() returns 1000 instead of 0
```solidity
    function getOhmEmissions() external view returns (uint256 emitted, uint256 removed) {
        uint256 currentPoolOhmShare = _getPoolOhmShare();

        if (ohmMinted > currentPoolOhmShare + ohmRemoved)
            emitted = ohmMinted - currentPoolOhmShare - ohmRemoved;
        else removed = currentPoolOhmShare + ohmRemoved - ohmMinted;
    }
```
## Impact
It will make the calculation involving ohmMinted incorrect.
## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L276-L277
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L392-L409
## Tool used

Manual Review

## Recommendation
```diff
    function withdraw(
        uint256 lpAmount_,
        uint256[] calldata minTokenAmounts_,
        bool claim_
    ) external onlyWhileActive nonReentrant returns (uint256) {
        // Liquidity vaults should always be built around a two token pool so we can assume
        // the array will always have two elements
        if (lpAmount_ == 0 || minTokenAmounts_[0] == 0 || minTokenAmounts_[1] == 0)
            revert LiquidityVault_InvalidParams();
        if (!_isPoolSafe()) revert LiquidityVault_PoolImbalanced();

        _withdrawUpdateRewardState(lpAmount_, claim_);

        totalLP -= lpAmount_;
        lpPositions[msg.sender] -= lpAmount_;

        // Withdraw OHM and pairToken from LP
        (uint256 ohmReceived, uint256 pairTokenReceived) = _withdraw(lpAmount_, minTokenAmounts_);

        // Reduce deposit values
        uint256 userDeposit = pairTokenDeposits[msg.sender];
        pairTokenDeposits[msg.sender] -= pairTokenReceived > userDeposit
            ? userDeposit
            : pairTokenReceived;
-       ohmMinted -= ohmReceived > ohmMinted ? ohmMinted : ohmReceived;
        ohmRemoved += ohmReceived > ohmMinted ? ohmReceived - ohmMinted : 0;
```