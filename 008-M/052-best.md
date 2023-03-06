ABA

medium

# Missing deduplication can result in vaults not being correctly removed using `removeVault` in `OlympusLiquidityRegistry`

## Summary

Missing deduplication can result in vaults not being correctly removed from liquidity registry in cases where a vault was mistakenly added more then once and then wanted to be removed.

## Vulnerability Detail

In `OlympusLiquidityRegistry.sol` tracked vaults can be added using `addVault` function and removed using `removeVault`.

When removing a vault, `removeVault` simply iterates through a list of known vaults, looking for it's address, removes it's first occurence and then exits.

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/modules/LQREG/OlympusLiquidityRegistry.sol#L42-L57


`addVault` does not check for duplicates when adding a vault, as such, any mistakenly added duplicated vault will remain there if the caller of `removeVault` is not aware of the duplication.
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/modules/LQREG/OlympusLiquidityRegistry.sol#L35-L40

## Impact

In the case of a mistakenly duplication vault, followed by just 1 remove, then the vault still remains in the liquidity registry.
Depending on how Olympus plans to use the registry, this may have consequences from low to severe, such as using a dead/compromised vault with further protocol logic. 

## Code Snippet

Add the following POC to `LQREG.t.sol`
```Solidity
    function testCorrectness_duplicatedVaultsAreNotRemoved() public {

        address problemVault = address(2);
        
        vm.startPrank(godmode);
        lqreg.addVault(address(1));   // vault #0
        lqreg.addVault(problemVault); // vault #1
        lqreg.addVault(problemVault); // vault #2
        lqreg.addVault(address(3));   // vault #3
        lqreg.addVault(address(4));   // vault #4
        vm.stopPrank();

        // Verify initial state
        assertEq(lqreg.activeVaultCount(), 5);
        assertEq(lqreg.activeVaults(1), problemVault);
        assertEq(lqreg.activeVaults(2), problemVault);

        vm.prank(godmode);
        lqreg.removeVault(problemVault);

        // Verify first vault instances was removed BUT the second instances still exists 
        assertEq(lqreg.activeVaultCount(), 4);
        assertTrue(lqreg.activeVaults(1) != problemVault);
        assertEq(lqreg.activeVaults(2), problemVault);
    }
```

## Tool used

Manual Review

## Recommendation

Either implement a deduplication logic in `addVault` or simply let the `removeVault` loop continue without the `break`. This will ensure that all duplications are removed and practically there will not by such a high number of vaults to make this solution too gas intensive.

Also move the emit event instruction after a vault has been removed in the loop. As it is, it is sent even if a random address is given but not found in the registry.