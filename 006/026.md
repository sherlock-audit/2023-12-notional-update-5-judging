Strong Leather Toad

high

# Rebalancing can lend beyond the debt cap

## Summary

Rebalancing can lend beyond the debt cap. If the debt cap is not properly enforced, users will have issues redeeming or withdrawing their prime cash.

## Vulnerability Detail

The max debt (`maxUnderlyingDebt`) is set to a percentage of the max supply (`maxUnderlyingSupply`), as shown below.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/pCash/PrimeSupplyCap.sol#L57

```solidity
File: PrimeSupplyCap.sol
55:         // If maxUnderlyingSupply or maxPrimeDebtUtilization is set to zero, there is no debt cap. The
56:         // debt cap is applied to prevent the supply cap from being locked up by extremely high utilization
57:         maxUnderlyingDebt = maxUnderlyingSupply
58:             .mul(s.maxPrimeDebtUtilization).div(uint256(Constants.PERCENTAGE_DECIMALS)); 
```

Assume that `maxUnderlyingSupply` is set to 1000 and `maxPrimeDebtUtilization` is set to 80%, then `maxUnderlyingDebt` will be 800.

If the `targetUtilization` is 90%, it will attempt to lend out money to the external market until 90% and accumulate the debt of 90% of the supply. 

Assume that the total supply is 1000 at this point. Then, it will aim/target to accumulate a debt of 900 by lending to the external market.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/ExternalLending.sol#L53

```solidity
File: ExternalLending.sol
36:     function getTargetExternalLendingAmount(
..SNIP..
49:         // The target amount to lend is based on a target "utilization" of the total prime supply. For example, for
50:         // a target utilization of 80%, if the prime cash utilization is 70% (totalPrimeSupply / totalPrimeDebt) then
51:         // we want to lend 10% of the total prime supply. This ensures that 20% of the totalPrimeSupply will not be held
52:         // in external money markets which run the risk of becoming unredeemable.
53:         int256 targetExternalUnderlyingLend = totalPrimeCashInUnderlying
54:             .mul(rebalancingTargetData.targetUtilization)
55:             .div(Constants.PERCENTAGE_DECIMALS)
56:             .sub(totalPrimeDebtInUnderlying);
```

At this point, the `totalUnderlyingDebt` is 900, and the `maxUnderlyingDebt` (debt cap) is 800, which shows that the debt cap has been bypassed.

## Impact

The purpose of the debt cap is to ensure the redeemability of prime cash and ensure sufficient liquidity for withdraws and liquidations. If the debt cap is not properly enforced, users or various core functionalities within the protocol will encounter issues redeeming or withdrawing their prime cash. For instance, users might not be able to withdraw their assets from the protocol due to insufficient liquidity, or liquidation cannot be carried out due to lack of liquidity, resulting in bad debt accumulating within the protocol and negatively affecting the protocol's solvency.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/pCash/PrimeSupplyCap.sol#L57

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/ExternalLending.sol#L53

## Tool used

Manual Review

## Recommendation

The root cause stems from the fact that during the rebalancing, it does not take into consideration the existing debt cap. It is recommended to include additional logic during the rebalancing process to ensure that it does not lend to the external market beyond the debt cap (`maxUnderlyingDebt`).