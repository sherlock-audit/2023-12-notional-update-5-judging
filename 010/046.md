Strong Leather Toad

medium

# Rebalance might be skipped even if the external lending is unhealthy

## Summary

The deviation between the target and current lending amount (`offTargetPercentage`) will be underestimated due to incorrect calculation. As a result, a rebalancing might be skipped even if the existing external lending is unhealthy.

## Vulnerability Detail

The formula used within the `_isExternalLendingUnhealthy` function below calculating the `offTargetPercentage` can be simplified as follows for the readability of this issue.

$$
offTargetPercentage = \frac{\mid currentExternalUnderlyingLend - targetAmount \mid}{currentExternalUnderlyingLend + targetAmount} \times 100\%
$$

Assume that the `targetAmount` is 100 and `currentExternalUnderlyingLend` is 90. The off-target percentage will be 5.26%, which is incorrect.

```solidity
offTargetPercentage = abs(90 - 100) / (100 + 90) = 10 / 190 = 0.0526 = 5.26%
```

The correct approach is to calculate the off-target percentages as a ratio of the difference to the target:

$$
offTargetPercentage = \frac{\mid currentExternalUnderlyingLend - targetAmount \mid}{targetAmount} \times 100\%
$$

```solidity
offTargetPercentage = abs(90 - 100) / (100) = 10 / 100 = 0.0526 = 10%
```

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L425

```solidity
File: TreasuryAction.sol
405:     function _isExternalLendingUnhealthy(
406:         uint16 currencyId,
407:         IPrimeCashHoldingsOracle oracle,
408:         PrimeRate memory pr
409:     ) internal view returns (bool isExternalLendingUnhealthy, OracleData memory oracleData, uint256 targetAmount) {
410:         oracleData = oracle.getOracleData(); 
411: 
412:         RebalancingTargetData memory rebalancingTargetData =
413:             LibStorage.getRebalancingTargets()[currencyId][oracleData.holding]; 
414:         PrimeCashFactors memory factors = PrimeCashExchangeRate.getPrimeCashFactors(currencyId); 
415:         Token memory underlyingToken = TokenHandler.getUnderlyingToken(currencyId); 
416: 
417:         targetAmount = ExternalLending.getTargetExternalLendingAmount(
418:             underlyingToken, factors, rebalancingTargetData, oracleData, pr
419:         ); 
420: 
421:         if (oracleData.currentExternalUnderlyingLend == 0) { 
422:             // If this is zero then there is no outstanding lending.
423:             isExternalLendingUnhealthy = false; 
424:         } else {
425:             uint256 offTargetPercentage = oracleData.currentExternalUnderlyingLend.toInt() 
426:                 .sub(targetAmount.toInt()).abs()
427:                 .toUint()
428:                 .mul(uint256(Constants.PERCENTAGE_DECIMALS))
429:                 .div(targetAmount.add(oracleData.currentExternalUnderlyingLend)); 
430:             
431:             // prevent rebalance if change is not greater than 1%, important for health check and avoiding triggering
432:             // rebalance shortly after rebalance on minimum change
433:             isExternalLendingUnhealthy = 
434:                 (targetAmount < oracleData.currentExternalUnderlyingLend) && (offTargetPercentage > 0); 
435:         }
436:     }
```

## Impact

The deviation between the target and current lending amount (`offTargetPercentage`) will be underestimated by approximately half the majority of the time. As a result, a rebalance intended to remediate the unhealthy external lending might be skipped since the code incorrectly assumes that it has not hit the off-target percentage. External lending beyond the target will affect the liquidity of the protocol, potentially resulting in withdrawal or liquidation, having issues executed due to insufficient liquidity.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L425

## Tool used

Manual Review

## Recommendation

Consider calculating the off-target percentages as a ratio of the difference to the target:

$$
offTargetPercentage = \frac{\mid currentExternalUnderlyingLend - targetAmount \mid}{targetAmount} \times 100\%
$$