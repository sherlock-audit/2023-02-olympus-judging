rvierdiiev

medium

# SingleSidedLiquidityVault.claimRewards doesn't update internal tokens reward

## Summary
SingleSidedLiquidityVault.claimRewards doesn't update internal tokens reward.
## Vulnerability Detail
SingleSidedLiquidityVault.claimRewards function doesn't call `_accumulateInternalRewards` and then `_updateInternalRewardState` for each internal reward token. However it does such for external token.
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L288-L310
```solidity
    function claimRewards() external onlyWhileActive nonReentrant {
        uint256 numInternalRewardTokens = internalRewardTokens.length;
        uint256 numExternalRewardTokens = externalRewardTokens.length;


        uint256[] memory accumulatedRewards = _accumulateExternalRewards();


        for (uint256 i; i < numInternalRewardTokens; ) {
            _claimInternalRewards(i);


            unchecked {
                ++i;
            }
        }


        for (uint256 i; i < numExternalRewardTokens; ) {
            _updateExternalRewardState(i, accumulatedRewards[i]);
            _claimExternalRewards(i);


            unchecked {
                ++i;
            }
        }
    }
```

Because of that when user claims, he can't receive everything that he should at the moment.

Of course, those funds are not lost for him. Updating will be called when someone [deposits](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L518) or [withdraws](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L575). 
But what if vault is not used for a long time? Then user doesn't have any other option in order to update his internal rewards and claim them.
## Impact
User can't claim full amount of internal rewards.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
You need to update internal rewards in same way as it's done for external tokens.