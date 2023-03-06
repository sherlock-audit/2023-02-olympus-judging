cducrest-brainbot

high

# Impermanent loss different from stated in docs

## Summary

The docs state that: "Users of SSLVs will experience identicaly impermanent loss (in dollar terms) as if they had split their pair token deposit into 50% OHM - 50% pair token and LP'd." This is not true based on code review.

## Vulnerability Detail

The user deposits pair token into the vault, and a position is opened with 50/50 usd value split in OHM / pair token. If the price of pair token increases compared to OHM, the value in the pool of pair token will increase. Arbitrageurs will stabilize the pool and bring it back to a value equilibrium. The value of OHM in the pool will thus increase and the value of pair token decrease as a result of arbitrage.

However, during withdraw the user will only receive pair token back, he will thus experience only the value decrease of pair token due to arbitrage and never profit from the value increase of OHM tokens.

Example: 

- A lone user puts 10 wstETH worth 1000$ in the pool. The pool is filled with 10_000$ worth of wstETH and OHM. The user started with a value of 10_000$.
- The price of wstETH increases to 1100$. Arbitrageurs will come and profit from this opportunity by withdrawing wstETH and depositing OHM. 
- The pool will reach equilibrium when there is 9.5347 wstETH in it (I skip the calculus)
- The pool will have a value of 10_488$ in both wstETH and OHM, due to the increase of wstETH price.
- If the user owned both the wstETH and OHM, he would have an impermenant loss of `price if he held (OHM and wstETH) - current withdraw price = 10 * 1100$ + 10_000 - 2 * 10_488 = 24$`
- However, since he can only withdraw the wstETH, he will experience an impermanent loss of `price if he held wstETH - current withdraw price = 10 * 1100$ - 10_488 = 512$`.

## Impact

Users experience humongous impermanent loss, which is unexpected from reading the docs. The OHM token intends to be stable in price (not regarding to dollar, but probably will), which should result in high impermanent loss for the users.

The problem will be reversed and the protocol will experience most of the loss if the price fluctuates in the other way (or the price of OHM fluctuates).

I set this issue as high, not because the docs do not match the implementation, but because I believe the developer team misunderstood the behaviour of the vault and I consider the design (notably of the withdraw function) flawed enough that users should not participate in this protocol.

## Code Snippet

withdraw function only send pair token out to user, based on current pool situation:

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L281

## Tool used

Manual Review

## Recommendation

Inform users of the risk / reconsider withdraw mechanics to properly reward the users.
