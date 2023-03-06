chaduke

medium

# DOS attack to getUsers()

## Summary
The getUser() function will return the whole array of users. It will run out of gas if a malicious user deposits will small amounts for a long list of wallet addresses. 

## Vulnerability Detail
The ``getUser()`` function needs to return the whole array of users in memory, which needs memory copy operation. As a result, when the list is too long, it will run out of gas. 

[https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L334-L336](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L334-L336)

Meanwhile, a malicious can deposit with small amount for a long list of wallet addresses to increase the length of the array ``users``.

[https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L187-L244](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L187-L244)

As a result, it creates an effective DOS to the ``getUsers()`` function.

## Impact
The function ``getUsers()`` is not useful anymore when there is a DOS attack.

## Code Snippet
See above

## Tool used
VSCode

Manual Review

## Recommendation
- Revise the function ``getUsers()`` into ``getUsers(from, to)`` so that we can retrieve the users within a range of indices. 

- Set up a minium deposit limit so that such attack is most costing. 
