0x52

medium

# setLimit check is incorrect

## Summary

setLimit compares the new limit directly to ohmMinted. This check is not accurate because when doing the actual limit checks it also takes into account the amount of ohmRemoved. 

## Vulnerability Detail

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L406-L409

The _canDeposit check is done to confirm that the deposit amount doesn't cause the amount of ohmMinted to exceed the SUM of the LIMIT and ohmRemoved. 

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L785-L788

When changing the limit it compares the new limit only against the amount minted but doesn't take into account ohmRemoved. The result is that the limit can't be constricted as much as it should be able to.

## Impact

setLimit check will prevent the contract from constricting the limit correctly 

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L785-L788

## Tool used

ChatGPT

## Recommendation

When checking the new limit, make sure to accoutn for ohmRemoved:

        function setLimit(uint256 limit_) external onlyRole("liquidityvault_admin") {
    -       if (limit_ < ohmMinted) revert LiquidityVault_InvalidParams();
    +       if (limit_ + ohmRemoved < ohmMinted) revert LiquidityVault_InvalidParams();
            LIMIT = limit_;
        }