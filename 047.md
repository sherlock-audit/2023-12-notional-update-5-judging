Strong Leather Toad

medium

# `lastRebalanceTimestampInSeconds` is not updated after rebalancing unhealthy external lending

## Summary

The `lastRebalanceTimestampInSeconds` is not updated after rebalancing unhealthy external lending. As a result, a rebalance is executed even if it is not required (no unhealthy external lending OR cooldown period has not passed yet since the last rebalance), leading to a loss of funds for Notional as it has to pay Gelato the fee for executing an unnecessary action.

## Vulnerability Detail

To check if the cooldown has passed, it relies on the `lastRebalanceTimestampInSeconds`.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L309

```solidity
File: TreasuryAction.sol
308:     /// @notice Returns when sufficient time has passed since the last rebalancing cool down.
309:     function _hasCooldownPassed(RebalancingContextStorage memory context) private view returns (bool) {
310:         return uint256(context.lastRebalanceTimestampInSeconds)
311:             .add(context.rebalancingCooldownInSeconds) < block.timestamp;
312:     }
```

The `lastRebalanceTimestampInSeconds` will only be updated when the `_updateOracleSupplyRate` function at Line 332 is executed. However, when there is an "unhealthy" rebalancing due to unhealthy external lending (unhealthy = current external lending exceeds the target amount), the rebalancing can be executed immediately even if the cooldown period has not passed yet (`hasCooldownPassed = false`).

In this case, the code did not update the `lastRebalanceTimestampInSeconds` to the current time when the "unhealthy" rebalancing is executed because the `_updateOracleSupplyRate` function will not be triggered. Thus, the time between the `lastRebalanceTimestampInSeconds` and the current time will remain the same (not reset back to zero), and it is possible for normal rebalancing to be executed again shortly after the "unhealthy" rebalance.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L332

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
336: 
337:         // External effects happen after the internal state has updated
338:         _executeRebalance(currencyId, oracle, pr, oracleData, targetAmount); 
339: 
340:         emit CurrencyRebalanced(currencyId, pr.supplyFactor.toUint(), oracleSupplyRate);
341:     }
```

## Impact

Notional uses Gelato bots for its rebalancing process. Whenever the rebalance is executed by the bots, Notional needs to compensate Gelato for the work performed. Due to this issue, a rebalance is executed even if it is not required (no unhealthy external lending OR cooldown period has not passed yet since the last rebalance), leading to a loss of funds for Notional as it has to pay Gelato the fee for executing an unnecessary action.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L332

## Tool used

Manual Review

## Recommendation

Update the `lastRebalanceTimestampInSeconds` to the current time after the normal and "unhealthy" rebalances are executed.