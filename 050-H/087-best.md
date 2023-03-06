0xbrett8571

high

# OHM Token contract allows unauthorized burning, causing an overflow or underflow.

## Summary
The functions `burnFromTreasury, burnFrom`, and burn in the Burner contract don't verify the subtraction of the burned tokens from the total supply of the OHM token will result in an underflow or an overflow, In both cases, the contract will not have the expected behavior, and it can compromise the funds of the contract.

## Vulnerability Detail
Both functions `burnFromTreasury` and `burnFrom` call the `burnOhm` function of `MINTR`, which burns the requested amount of OHM from the address that calls it, but these functions do not verify whether subtracting the burned tokens from the total supply of OHM will result in an underflow or an overflow. In both cases, the contract will not have the expected behavior, and it can compromise the funds of the contract.

The functions `burnFromTreasury(), burnFrom()`, and `burn()` use the `burnOhm()` function from the `MINTR` module to burn OHM tokens, this function does not verify if the subtraction of the burned tokens from the total supply of OHM will result in an underflow or an overflow.

For example, if the total supply of OHM is 1, and you call the `burn()` function to burn 2 OHM, an underflow will occur, and the result will be a large positive number, similarly, if the total supply of OHM is close to the maximum value that can be stored in the variable, and you burn a large amount of OHM, an overflow will occur, and the result will be a small negative number.

Exploit Scenario:
Someone with an attack intention can call the `burnFromTreasury, burnFrom`, or burn function and burn a large amount of OHM tokens without proper verification. The attacker can repeat this action until an underflow, or an overflow is caused, which can cause the contract to behave unexpectedly.

## Impact
It allows attackers to burn OHM tokens without proper verification, leading to an overflow or underflow, and an attacker can exploit this vulnerability to compromise the funds of the contract.

## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/Burner.sol#L101
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/Burner.sol#L119

## Tool used

Manual Review

## Recommendation
Modify the burn functions to ensure that the resulting total supply of OHM is always greater than or equal to zero before performing the burn operation by adding a `require` statement before the burn operation to check if the total supply will not underflow. If the total supply is less than the amount to burn, the `require` statement should revert the transaction.

For example, in the `burnFromTreasury` function, you can modify the code to add the following condition:
```solidity
uint256 totalSupply = ohm.totalSupply();
require(totalSupply >= amount_, "Burn amount exceeds total supply");
require(totalSupply - amount_ < totalSupply, "Invalid burn amount");
```
The first `require` statement checks if the amount to burn is less than or equal to the total supply. 
The second `require` statement checks if the resulting total supply after the burn operation will not exceed the maximum value.