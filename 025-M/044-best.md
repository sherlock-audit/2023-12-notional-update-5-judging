Strong Leather Toad

medium

# Rebalance will be delayed due to revert

## Summary

The rebalancing of unhealthy currencies will be delayed due to a revert, resulting in an excess of liquidity being lent out in the external market. This might affect the liquidity of the protocol, potentially resulting in withdrawal or liquidation having issues executed due to insufficient liquidity.

## Vulnerability Detail

Assume that Notional supports 5 currencies ($A, B, C, D, E$), and the Gelato bot is configured to call the `checkRebalance` function every 30 minutes.

Assume that the current market condition is volatile. Thus, the inflow and outflow of assets to Notional, utilization rate, and available liquidity at AAVE change frequently. As a result, the target amount that should be externally lent out also changes frequently since the computation of this value relies on the spot market information.

At T1, when the Gelato bot calls the `checkRebalance()` view function, it returns that currencies $A$, $B$, and $C$ are unhealthy and need to be rebalanced.

Shortly after receiving the execution payload from the `checkRebalance()`, the bot submits the rebalancing TX to the mempool for execution at T2.

When the rebalancing TX is executed at T3, one of the currencies (Currency $A$) becomes healthy. As a result, the require check at Line 326 will revert and the entire rebalancing transaction will be cancelled. Thus, currencies $B$ and $C$ that are still unhealthy at this point will not be rebalanced.

If this issue occurs frequently or repeatedly over a period of time, the rebalancing of unhealthy currencies will be delayed.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L326

```solidity
File: TreasuryAction.sol
315:     function _rebalanceCurrency(uint16 currencyId, bool useCooldownCheck) private { 
316:         RebalancingContextStorage memory context = LibStorage.getRebalancingContext()[currencyId]; 
317:         // Accrues interest up to the current block before any rebalancing is executed
318:         IPrimeCashHoldingsOracle oracle = PrimeCashExchangeRate.getPrimeCashHoldingsOracle(currencyId); 
319:         PrimeRate memory pr = PrimeRateLib.buildPrimeRateStateful(currencyId); 
320: 
321:         bool hasCooldownPassed = _hasCooldownPassed(context); 
322:         (bool isExternalLendingUnhealthy, OracleData memory oracleData, uint256 targetAmount) = 
323:             _isExternalLendingUnhealthy(currencyId, oracle, pr); 
324: 
325:         // Cooldown check is bypassed when the owner updates the rebalancing targets
326:         if (useCooldownCheck) require(hasCooldownPassed || isExternalLendingUnhealthy); 
```

## Impact

The rebalancing of unhealthy currencies will be delayed, resulting in an excess of liquidity being lent out to the external market. This might affect the liquidity of the protocol, potentially resulting in withdrawal or liquidation having issues executed due to insufficient liquidity.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L326

## Tool used

Manual Review

## Recommendation

If one of the currencies becomes healthy when the rebalance TX is executed, consider skipping this currency and move on to execute the rebalance on the rest of the currencies that are still unhealthy.

```diff
function _rebalanceCurrency(uint16 currencyId, bool useCooldownCheck) private { 
	RebalancingContextStorage memory context = LibStorage.getRebalancingContext()[currencyId]; 
	// Accrues interest up to the current block before any rebalancing is executed
	IPrimeCashHoldingsOracle oracle = PrimeCashExchangeRate.getPrimeCashHoldingsOracle(currencyId); 
	PrimeRate memory pr = PrimeRateLib.buildPrimeRateStateful(currencyId); 

	bool hasCooldownPassed = _hasCooldownPassed(context); 
	(bool isExternalLendingUnhealthy, OracleData memory oracleData, uint256 targetAmount) = 
		_isExternalLendingUnhealthy(currencyId, oracle, pr); 

	// Cooldown check is bypassed when the owner updates the rebalancing targets
-	if (useCooldownCheck) require(hasCooldownPassed || isExternalLendingUnhealthy);
+	if (useCooldownCheck && !hasCooldownPassed && !isExternalLendingUnhealthy) return;
```