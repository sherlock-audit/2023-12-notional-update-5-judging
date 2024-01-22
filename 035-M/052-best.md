Short Candy Salmon

medium

# getTargetExternalLendingAmount() targetAmount may far less than the correct value

## Summary
When calculating `ExternalLending.getTargetExternalLendingAmount()`,
it restricts `targetAmount`  greater than `oracleData.maxExternalDeposit`.
However, it does not take into account that `oracleData.maxExternalDeposit` includes the protocol deposit `currentExternalUnderlyingLend` 
This may result in the returned quantity being far less than the correct quantity.

## Vulnerability Detail
in `getTargetExternalLendingAmount()` 
It restricts `targetAmount` greater than `oracleData.maxExternalDeposit`.
```solidity
    function getTargetExternalLendingAmount(
        Token memory underlyingToken,
        PrimeCashFactors memory factors,
        RebalancingTargetData memory rebalancingTargetData,
        OracleData memory oracleData,
        PrimeRate memory pr
    ) internal pure returns (uint256 targetAmount) {
...

        targetAmount = SafeUint256.min(
            // totalPrimeCashInUnderlying and totalPrimeDebtInUnderlying are in 8 decimals, convert it to native
            // token precision here for accurate comparison. No underflow possible since targetExternalUnderlyingLend
            // is floored at zero.
            uint256(underlyingToken.convertToExternal(targetExternalUnderlyingLend)),
            // maxExternalUnderlyingLend is limit enforced by setting externalWithdrawThreshold
            // maxExternalDeposit is limit due to the supply cap on external pools
@>          SafeUint256.min(maxExternalUnderlyingLend, oracleData.maxExternalDeposit)
        );
```
this is : `targetAmount = min(targetExternalUnderlyingLend, maxExternalUnderlyingLend, oracleData.maxExternalDeposit)`

The problem is that when calculating `oracleData.maxExternalDeposit`, it does not exclude the existing deposit `currentExternalUnderlyingLend` of the current protocol.

For example:
`currentExternalUnderlyingLend = 100`
`targetExternalUnderlyingLend = 100`
`maxExternalUnderlyingLend = 10000`
`oracleData.maxExternalDeposit = 0`        (All AAVE deposits include the current deposit `currentExternalUnderlyingLend`)

If according to the current calculation result: `targetAmount=0`, this will result in needing to withdraw `100`.  (currentExternalUnderlyingLend - targetAmount)

In fact, only when the calculation result needs to increase the `deposit` (targetAmount > currentExternalUnderlyingLend), it needs to be restricted by `maxExternalDeposit`.

The correct one should be neither deposit nor withdraw, that is, `targetAmount=currentExternalUnderlyingLend = 100`.

## Impact

A too small `targetAmount` will cause the withdrawal of deposits that should not be withdrawn, damaging the interests of the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/ExternalLending.sol#L89C1-L97C11

## Tool used

Manual Review

## Recommendation

Only when `targetAmount > currentExternalUnderlyingLend` is a deposit needed, it should be considered that it cannot exceed `oracleData.maxExternalDeposit`

```diff
    function getTargetExternalLendingAmount(
        Token memory underlyingToken,
        PrimeCashFactors memory factors,
        RebalancingTargetData memory rebalancingTargetData,
        OracleData memory oracleData,
        PrimeRate memory pr
    ) internal pure returns (uint256 targetAmount) {
...

-        targetAmount = SafeUint256.min(
-            // totalPrimeCashInUnderlying and totalPrimeDebtInUnderlying are in 8 decimals, convert it to native
-            // token precision here for accurate comparison. No underflow possible since targetExternalUnderlyingLend
-            // is floored at zero.
-            uint256(underlyingToken.convertToExternal(targetExternalUnderlyingLend)),
-            // maxExternalUnderlyingLend is limit enforced by setting externalWithdrawThreshold
-            // maxExternalDeposit is limit due to the supply cap on external pools
-            SafeUint256.min(maxExternalUnderlyingLend, oracleData.maxExternalDeposit)
-        );

+       targetAmount = SafeUint256.min(uint256(underlyingToken.convertToExternal(targetExternalUnderlyingLend)),maxExternalUnderlyingLend);
+       if (targetAmount > oracleData.currentExternalUnderlyingLend) { //when deposit ,  must check maxExternalDeposit
+            uint256 forDeposit = targetAmount - oracleData.currentExternalUnderlyingLend;
+            if (forDeposit > oracleData.maxExternalDeposit) {
+                targetAmount = targetAmount.sub(
+                    forDeposit - oracleData.maxExternalDeposit
+                );                
+            }
+        }
```
