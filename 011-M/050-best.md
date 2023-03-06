rvierdiiev

medium

# SingleSidedLiquidityVault _canDeposit and getMaxDeposit are checking maximum amount in different ways

## Summary
SingleSidedLiquidityVault _canDeposit and getMaxDeposit are checking maximum amount in different ways
## Vulnerability Detail
`_canDeposit` is used in order to check if user can deposit provided amount of ohm.
 https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L406-L409
```solidity
    function _canDeposit(uint256 amount_) internal view virtual returns (bool) {
        if (amount_ + ohmMinted > LIMIT + ohmRemoved) revert LiquidityVault_LimitViolation();
        return true;
    }
```
As you can see user can deposit if his ohm amount + total ohmMinted is less than LIMIT + ohmRemoved.
So max amount that is allowed here is `uint256 maxOhmAmount = LIMIT + ohmRemoved - ohmMinted`.

Now let's check `getMaxDeposit` function.
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L318-L330
```solidity
    function getMaxDeposit() public view returns (uint256) {
        uint256 currentPoolOhmShare = _getPoolOhmShare();
        uint256 emitted;


        // Calculate max OHM mintable amount
        if (ohmMinted > currentPoolOhmShare) emitted = ohmMinted - currentPoolOhmShare;
        uint256 maxOhmAmount = LIMIT + ohmRemoved - ohmMinted - emitted;


        // Convert max OHM mintable amount to pair token amount
        uint256 ohmPerPairToken = _valueCollateral(1e18); // OHM per 1 pairToken
        uint256 pairTokenDecimalAdjustment = 10**pairToken.decimals();
        return (maxOhmAmount * pairTokenDecimalAdjustment) / ohmPerPairToken;
    }
```
As you can see in this function `uint256 maxOhmAmount = LIMIT + ohmRemoved - ohmMinted - emitted`. And you can notice that is almost same formula as in `_canDeposit`, but we have additional param here `emitted`, which can decrease maximum amount.

In case if `_canDeposit` function calculates incorrectly, then it allow users to deposit more, than protocol allows.
In case if `getMaxDeposit` calculates incorrectly, which will be used by frontend and integration contracts, users and contracts that wants to integrate with protocol will receive wrong information about max deposit amount, and can have integration issues.
## Impact
Max deposit amount is calculated in different ways.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
You need to use same way in both functions.