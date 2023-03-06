tsvetanovv

medium

# Missing deadline check in `deposit()` function

## Summary
In `SingleSidedLiquidityVault.sol` we have [deposit()](https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L187). This function missing deadline check.

## Vulnerability Detail
Missing deadline checks allow pending transactions to be maliciously executed in the future. You need to add a deadline parameter to all functions which potentially perform a swap on the user's behalf.

## Impact
Without deadline parameter, as a consequence, users can have their operations executed at unexpected times, when the market conditions are unfavorable.

## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L187
```solidity
function deposit(uint256 amount_, uint256 slippageParam_) 
        external
        onlyWhileActive
        nonReentrant
        returns (uint256 lpAmountOut)
    {

        // If this is a new user, add them to the users array in case we need to migrate
        // their state in the future

        if (!_hasDeposited[msg.sender]) {
            _hasDeposited[msg.sender] = true;
            users.push(msg.sender); //@audit possible DOS loop
        }

        // Calculate amount of OHM to borrow
        uint256 ohmToBorrow = _valueCollateral(amount_);

        // Cache pair token and OHM balance before deposit
        uint256 pairTokenBalanceBefore = pairToken.balanceOf(address(this));
        uint256 ohmBalanceBefore = ohm.balanceOf(address(this));

        // The pool being imbalanced is less of a concern here on deposit than on withdrawal,
        // but in the event the frontend miscalculates the expected LP amount to receive, we want
        // to reduce the risk of entering a manipulated pool at a bad price
        if (!_isPoolSafe()) revert LiquidityVault_PoolImbalanced();
        if (!_canDeposit(ohmToBorrow)) revert LiquidityVault_LimitViolation();

        _depositUpdateRewardState();

        // Gather tokens for deposit
        pairToken.safeTransferFrom(msg.sender, address(this), amount_); 
        _borrow(ohmToBorrow);

  
        uint256 lpReceived = _deposit(ohmToBorrow, amount_, slippageParam_);

  
        // Calculate amount of pair tokens and OHM unused in deposit
        uint256 unusedPairToken = pairToken.balanceOf(address(this)) - pairTokenBalanceBefore;
        uint256 unusedOhm = ohm.balanceOf(address(this)) - ohmBalanceBefore;


        // Return unused pair tokens to user
        if (unusedPairToken > 0) pairToken.safeTransfer(msg.sender, unusedPairToken);

  
        // Burn unused OHM

        if (unusedOhm > 0) _repay(unusedOhm);

  
        uint256 pairTokenUsed = amount_ - unusedPairToken;
        uint256 ohmUsed = ohmToBorrow - unusedOhm;

  
        ohmMinted += ohmUsed;
        totalLP += lpReceived;

  
        pairTokenDeposits[msg.sender] += pairTokenUsed;
        lpPositions[msg.sender] += lpReceived;

  
        // Update user's reward debts
        _depositUpdateRewardDebts(lpReceived);

        emit Deposit(msg.sender, pairTokenUsed, ohmUsed);

    }
```

## Tool used

Manual Review

## Recommendation
Introduce a `deadline` parameter in `deposit()` function.