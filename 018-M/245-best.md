gogo

medium

# No check for internal and external rewards tokens being the same

## Summary

A comment in SingleSidedLiquidityVault states that there shouldn't be same internal and external reward tokens but there is no check performing this validation.

## Vulnerability Detail

A comment in SingleSidedLiquidityVault states that there shouldn't be same internal and external reward tokens but there is no check performing this validation.


## Impact

Unexpected behaviour when claiming rewards

## Code Snippet

```solidity
///         - No internal reward token should also be an external reward token
```

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L25

```solidity
    /// @notice                    Adds a new internal reward token to the contract
    /// @param  token_             The address of the reward token
    /// @param  rewardsPerSecond_  The amount of reward tokens to distribute per second
    /// @param  startTimestamp_    The timestamp at which to start distributing rewards
    /// @dev                       This function can only be accessed by the liquidityvault_admin role
    function addInternalRewardToken(
        address token_,
        uint256 rewardsPerSecond_,
        uint256 startTimestamp_
    ) external onlyRole("liquidityvault_admin") {
        InternalRewardToken memory newInternalRewardToken = InternalRewardToken({
            token: token_,
            decimalsAdjustment: 10**ERC20(token_).decimals(),
            rewardsPerSecond: rewardsPerSecond_,
            lastRewardTime: block.timestamp > startTimestamp_ ? block.timestamp : startTimestamp_,
            accumulatedRewardsPerShare: 0
        });

        internalRewardTokens.push(newInternalRewardToken);
    }

    /// @notice                 Removes an internal reward token from the contract
    /// @param  id_             The index of the reward token to remove
    /// @param  token_          The address of the reward token to remove
    /// @dev                    This function can only be accessed by the liquidityvault_admin role
    function removeInternalRewardToken(uint256 id_, address token_)
        external
        onlyRole("liquidityvault_admin")
    {
        if (internalRewardTokens[id_].token != token_) revert LiquidityVault_InvalidRemoval();

        // Delete reward token from array by swapping with the last element and popping
        internalRewardTokens[id_] = internalRewardTokens[internalRewardTokens.length - 1];
       internalRewardTokens.pop();
    }
```

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L669-L703

## Tool used

Manual Review

## Recommendation

Check if internal token is already present in external tokens array and vice versa.
