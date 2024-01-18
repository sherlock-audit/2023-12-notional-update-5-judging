Strong Leather Toad

high

# Rebalance is executed even if the change is less than 1%

## Summary

The protocol highlighted that rebalancing should be prevented if the change is less than 1%. However, due to an error, a rebalance is executed even if the change is less than 1%, resulting in a rebalance being executed when it is actually not required. Since Notional needs to compensate Gelato for the work performed, it leads to a loss of funds for Notional as it has to pay Gelato the fee for executing an unnecessary action.

## Vulnerability Detail

Per the comment on Line 431 below, it mentioned that rebalancing should be prevented if the change is not greater than 1%. This means that only if the change is 1%, 1.01%, 2%, or larger, the rebalancing will be executed.

The reason is to avoid triggering rebalance shortly after rebalance on minimum change. The control is in place to restrict the rebalancing bots from carrying out rebalancing action when the deviation is minimal so as to avoid wasting too much gas performing an action that does not have a material impact on the protocol. For instance, if the deviation from the current lending amount and target is only 1 wei, performing a rebalancing would be unnecessary as it does not lead to a significant result.

However, the inequality `(offTargetPercentage > 0)` in Line 434 is incorrect as it means that any change larger than zero can be executed. As such, even if the change is infinitely small, close to zero (e.g., 0.0000000001%), the inequality (offTargetPercentage > 0) will still be evaluated as true, and the external lending is considered unhealthy, and the rebalance can proceed immediately.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L434

```solidity
File: TreasuryAction.sol
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
```

## Impact

Notional uses Gelato bots for its rebalancing process. Whenever the rebalance is executed by the bots, Notional needs to compensate Gelato for the work performed.

The Gelato bots will call the `checkRebalance` view function to check if rebalancing can be executed. In this case, the `checkRebalance` function will always mark the external lending as unhealthy whenever there is a tiny value change and wrongly indicate a rebalance is needed when it is actually not required, leading to a loss of funds for Notional as it has to pay Gelato the fee for executing an unnecessary action.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L434

## Tool used

Manual Review

## Recommendation

Consider implementing the following fix to ensure that rebalance is only executed if the deviation is more than 1%.

```diff
isExternalLendingUnhealthy = 
-    (targetAmount < oracleData.currentExternalUnderlyingLend) && (offTargetPercentage > 0);
+    (targetAmount < oracleData.currentExternalUnderlyingLend) && (offTargetPercentage >= 1);
```