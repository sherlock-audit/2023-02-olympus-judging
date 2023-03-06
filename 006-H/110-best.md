0x52

high

# Adversary can economically exploit wstETHLiquidityVault

## Summary

Adversary can profit off of the single sided liquidity vault by depositing, buying OHM, withdrawing then dumping the profited OHM. This attack remains profitable regardless of the value of `THRESHOLD`.

## Vulnerability Detail

SingleSidedLiquidityVault#deposit allows a user to specify the amount of wstETH they wish to deposit into the vault. The vault then mints the proper amount of OHM to match this, then deposits both into the wstETH/OHM liquidity pool on Balancer. If the price of OHM changes between deposit and withdrawal, the vault will effectively eat the IL caused by the movement. If the price decreases then the vault will burn more OHM than minted. If the price increases then the vault will burn less OHM than minted. This discrepancy can be exploited by malicious users to profit at the expense of the vault.

First we will outline the flow of the attack then run through the numbers:
1. Deposit wstETH, which causes the vault to mint OHM as a counter-asset
2. Buy OHM from the liquidity pool making sure to not go outside the price threshold to trigger the isPoolSafe check
3. Withdraw wstETH
4. Sell acquired OHM for a profit

Now we can crunch the numbers to prove that this is profitable:

The only assumption we need to make is the price of OHM/wstETH which for simplicity we will assume is 1:1.

Balances before attack:
Liquidity: 80 OHM 80 wstETH
Adversary: 20 wstETH

Balances after adversary has deposited to the pool:
Liquidity: 100 OHM 100 wstETH
Adversary: 0 wstETH

Balances after adversary sells wstETH for OHM (1% movement in price):
Liquidity: 99.503 OHM 100.498 wstETH
Adversary: 0.496 OHM -0.498 wstETH

Balances after adversary removes their liquidity:
Liquidity: 79.602 OHM 80.399 wstETH
Adversary: 0.496 OHM 19.7 wstETH 

Balances after selling profited OHM:
Liquidity: 80.099 OHM 79.9 wstETH 
Adversary: 20.099 wstETH

We can see that the adversary will gain wstETH for each time they loop this through attack. The profit being made i For simplicity I have only walked through a single direction attack but the adversary could easily drop the price to the lower threshold then start the attack to gain a larger amount of wstETH.

No matter how tight the threshold is set it is impossible to make this kind of attack unprofitable. Tighter thresholds just increases the amount of capital required to make it profitable. Another issue is that the THRESHOLD value can only get so small before the it starts causing random reverts for legitimate users.

For additional context, the fee charged by the pool only slightly impacts the profitability of this attack. Since the attacker only needs to manipulate the price within the threshold, fees scale linearly with THRESHOLD and therefore don't change the profitability of the attack.

## Impact

Vault can be exploited for a nearly unlimited amount of OHM

## Code Snippet

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L187-L244

## Tool used

ChatGPT

## Recommendation

The only mechanism I can think of to prevent this is to add a withdraw/deposit fee to the vault