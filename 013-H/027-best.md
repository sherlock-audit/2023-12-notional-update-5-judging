Strong Leather Toad

high

# Oracle supply rate is vulnerable to manipulation

## Summary

Malicious users could manipulate the oracle rate to perform market/price manipulation, which could lead to loss of funds. 

For instance, malicious users could depress the rate, which causes portfolio values to decrease and liquidate the victims. On the other hand, malicious users could increase the rate to inflate their portfolio values to over-borrow, resulting in the protocol incurring bad debt and affecting the insolvency of the protocol.

## Vulnerability Detail

Based on the test script in the audit repository, the `rebalancingCooldownInSeconds` is set to around 5 hours. Assume that the rebalancing cooldown is set to 5 hours in this report.

With the rebalancing cooldown in place, this means that a rebalance can only be executed once every 5 hours unless the external lending is unhealthy, which in this case can be executed immediately. The rebalance is triggered by the rebalancing bots.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L309

```solidity
File: TreasuryAction.sol
308:     /// @notice Returns when sufficient time has passed since the last rebalancing cool down.
309:     function _hasCooldownPassed(RebalancingContextStorage memory context) private view returns (bool) {
310:         return uint256(context.lastRebalanceTimestampInSeconds)
311:             .add(context.rebalancingCooldownInSeconds) < block.timestamp;
312:     }
```

During the rebalancing, the oracle supply rate will be updated if the cool-down has passed. Refer to Line 328 below.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L328

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
327: 
328:         // Updates the oracle supply rate as well as the last cooldown timestamp. Only update the oracle supply rate
329:         // if the cooldown has passed. If not, the oracle supply rate won't change.
330:         uint256 oracleSupplyRate = pr.oracleSupplyRate; 
331:         if (hasCooldownPassed) {
332:             oracleSupplyRate = _updateOracleSupplyRate(
333:                 currencyId, pr, context.previousSupplyFactorAtRebalance, context.lastRebalanceTimestampInSeconds
334:             );
335:         } 
```

Within the `_updateOracleSupplyRate ` function below, the comment at Line 343 mentioned that it is important to use a TWAP oracle here to ensure that this fCash is not subject to market manipulation. However, upon reviewing the `updateRateOracle ` function, it was observed that it does not implement any TWAP. The oracle supply rate is computed based on the current spot value of the `pr.supplyFactor`, which can be manipulated.

The oracle supply rate can be simplified as follows (ignoring the precision accuracy and ordering for simplicity's sake):

```solidity
interestRate = pr.supplyFactor / previousSupplyFactorAtRebalance

timePassSinceLastRebalanceInSeconds = block.timestamp. - lastRebalanceTimestampInSeconds
oracleSupplyRate = (interestRate / timePassSinceLastRebalanceInSeconds) * Constants.YEAR // To compute the annualize rate
```

If one could inflate the `pr.supplyFactor`, the oracle supply rate will be inflated.

Assuming that the cooldown has passed and the bots submit a rebalance TX, a malicious user could front-run the rebalance TX and inflate the `pr.supplyFactor`. When the rebalance TX is executed, the inflated oracle supply rate will immediately be stored in the storage at Line 372 below.

The attacker back-runs the rebalance TX to exploit the inflated/manipulated oracle rate and subsequently potentially use the ill-gain assets to repay the flash loan if it is used to manipulate the fCash market earlier. Flash-loan is not required if the attacker is well-funded.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L346

```solidity
File: TreasuryAction.sol
343:     /// @notice Updates the oracle supply rate for Prime Cash. This oracle supply rate is used to value fCash that exists
344:     /// below the 3 month fCash tenor. This is not a common situation but it is important to use a TWAP oracle here to
345:     /// ensure that this fCash is not subject to market manipulation.
346:     function _updateOracleSupplyRate( 
347:         uint16 currencyId,
348:         PrimeRate memory pr,
349:         uint256 previousSupplyFactorAtRebalance,
350:         uint256 lastRebalanceTimestampInSeconds
351:     ) private returns (uint256 oracleSupplyRate) {
352:         // If previous supply factor at rebalance == 0, then it is the first rebalance and the
353:         // oracle supply rate will be left as zero. The previous supply factor will
354:         // be set to the new factors.supplyFactor in the code below.
355:         if (previousSupplyFactorAtRebalance != 0) { 
356:             // The interest rate is the rate of increase of the supply factor scaled up to a
357:             // year time period. Therefore the calculation is:
358:             //  ((supplyFactor / prevSupplyFactorAtRebalance) - 1) * (year / timeSinceLastRebalance)
359:             uint256 interestRate = pr.supplyFactor.toUint() 
360:                 .mul(Constants.SCALAR_PRECISION)
361:                 .div(previousSupplyFactorAtRebalance)
362:                 .sub(Constants.SCALAR_PRECISION) 
363:                 .div(uint256(Constants.RATE_PRECISION)); 
364: 
365: 
366:             oracleSupplyRate = interestRate
367:                 .mul(Constants.YEAR)
368:                 .div(block.timestamp.sub(lastRebalanceTimestampInSeconds)); 
369:         }
370: 
371:         mapping(uint256 => PrimeCashFactorsStorage) storage p = LibStorage.getPrimeCashFactors();
372:         p[currencyId].oracleSupplyRate = oracleSupplyRate.toUint32(); 
373: 
374:         mapping(uint16 => RebalancingContextStorage) storage c = LibStorage.getRebalancingContext();
375:         c[currencyId].lastRebalanceTimestampInSeconds = block.timestamp.toUint40();
376:         c[currencyId].previousSupplyFactorAtRebalance = pr.supplyFactor.toUint().toUint128(); 
377:     }
```

## Impact

Malicious users could manipulate the oracle rate to perform market/price manipulation, which could lead to loss of funds. 

For instance, malicious users could depress the rate, which causes portfolio values to decrease and liquidate the victims. On the other hand, malicious users could increase the rate to inflate their portfolio values to over-borrow, resulting in the protocol incurring bad debt and affecting the insolvency of the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L309

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L328

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L346

## Tool used

Manual Review

## Recommendation

Implement a TWAP similar to the one within the `InterestRateCurve.updateRateOracle` function.