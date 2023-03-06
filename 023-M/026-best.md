chaduke

medium

# addExternalRewardToken() fails to set the inititial balance to the correct value.

## Summary
``addExternalRewardToken()`` fails to set the initial balance. It assumes a zero balance, which might not be true - for example, the external reward was removed and then added back.

The impact would be the wrong calculation of external rewards when the initial balance is not set right. 

## Vulnerability Detail

1. The ``addExternalRewardToken()`` function set the newRewardToken.lastBlance = 0 for a newly added external reward token. 

[https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L708-L717](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L708-L717)

2. Meanwhile, it relies on this ``lastBalance`` to check how much reward has been accumulated externally. 

[https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L192-L216](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/WstethLiquidityVault.sol#L192-L216)

3. As a result, if the initial balance is not set right, the wrong award will be calculated. 

4. The ``addExternalRewardToken()`` always set the initial balance to 0, even when it is NOT zero.  For example, the external reward was removed and then added back later, in this case, the initial balance might not be zero. 

## Impact
Due to possible wrong initial balance setup, reward can be calculated wrongly.

## Code Snippet
See above

## Tool used
VScode

Manual Review

## Recommendation
We will always need to check the real balance and set it correctly rather than assuming it is zero.

```diff
 function addExternalRewardToken(address token_) external onlyRole("liquidityvault_admin") {

+     uint lastBal = ERC20(token_).balanceOf(address(this));

        ExternalRewardToken memory newRewardToken = ExternalRewardToken({
            token: token_,
            decimalsAdjustment: 10**ERC20(token_).decimals(),
            accumulatedRewardsPerShare: 0,
-            lastBalance: 0
+           lastBalance: lastBal
        });

        externalRewardTokens.push(newRewardToken);
    }

```
