Bauer

medium

# If the vault is not active,user's assets will be frozen in the protocol

## Summary
 If the vault is not active ,user's assets will be frozen in the protocol. The same with claiming rewards.

## Vulnerability Detail
```solidity
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
        ohmMinted -= ohmReceived > ohmMinted ? ohmMinted : ohmReceived;
        ohmRemoved += ohmReceived > ohmMinted ? ohmReceived - ohmMinted : 0;

        // Return assets
        _repay(ohmReceived);
        pairToken.safeTransfer(msg.sender, pairTokenReceived);

        emit Withdraw(msg.sender, pairTokenReceived, ohmReceived);
        return pairTokenReceived;
    }

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

The protocol allows users to  deposit pair tokens, mints OHM against the deposited pair tokens and deposit the
 pair token and OHM into a liquidity pool and receives LP tokens in return. It also supports withdraw pair tokens and OHM from a liquidity pool, returns any received pair tokens to the user, and burns any received OHM. As the code above, we can find that the withdrawal function has a modifier ```onlyWhileActive```. If the vault is not active ,user's assets will be frozen in the protocol.
The same with claiming rewards.
## Impact
If the vault is not active ,user's assets will be frozen in the protocol.
## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L252-L285
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L288-L310

## Tool used

Manual Review

## Recommendation
Remove ```onlyWhileActive``` for functions ```withdraw``` and ```claimRewards```.
