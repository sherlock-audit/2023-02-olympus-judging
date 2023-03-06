0x52

medium

# Burner.sol will leave unsafe approval to previous MINTR if it changes

## Summary

Burner#configureDependencies allows the contract to pull a fresh reference to the TRSRY MINTR and ROLES contracts. In the event that the MINTR changes, the old MINTR contract will be left with an unsafe allowance to the old MINTR. If that contract is being replaced because it was compromised or buggy then it would be able to abuse the allowance left to it.

## Vulnerability Detail

See summary.

## Impact

Previous MINTR contract can abuse leftover allowance

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/Burner.sol#L58-L70

## Tool used

ChatGPT

## Recommendation

Contract should remove allowance to current MINTR before changing it:

    +   if (MINTR != address(0)) {
    +       ohm.safeApprove(address(MINTR), 0)
    +   }

        TRSRY = TRSRYv1(getModuleAddress(dependencies[0]));
        MINTR = MINTRv1(getModuleAddress(dependencies[1]));
        ROLES = ROLESv1(getModuleAddress(dependencies[2]));