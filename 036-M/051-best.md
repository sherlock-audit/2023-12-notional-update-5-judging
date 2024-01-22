Short Candy Salmon

medium

# getTargetExternalLendingAmount() when targetUtilization == 0  no check whether enough externalUnderlyingAvailableForWithdraw

## Summary
in `getTargetExternalLendingAmount()`
When `targetUtilization == 0`, it directly returns `targetAmount=0`.
It lacks the judgment of whether there is enough `externalUnderlyingAvailableForWithdraw`.
This may cause `_rebalanceCurrency()` to `revert` due to insufficient balance for `withdraw`.

## Vulnerability Detail

when `setRebalancingTargets()`  , we can setting all the targets to zero to immediately exit money
it will call `_rebalanceCurrency() -> _isExternalLendingUnhealthy() -> getTargetExternalLendingAmount()` 
```solidity
    function getTargetExternalLendingAmount(
        Token memory underlyingToken,
        PrimeCashFactors memory factors,
        RebalancingTargetData memory rebalancingTargetData,
        OracleData memory oracleData,
        PrimeRate memory pr
    ) internal pure returns (uint256 targetAmount) {
        // Short circuit a zero target
@>      if (rebalancingTargetData.targetUtilization == 0) return 0;

....
        if (targetAmount < oracleData.currentExternalUnderlyingLend) {
            uint256 forRedemption = oracleData.currentExternalUnderlyingLend - targetAmount;
            if (oracleData.externalUnderlyingAvailableForWithdraw < forRedemption) {
                // increase target amount so that redemptions amount match externalUnderlyingAvailableForWithdraw
                targetAmount = targetAmount.add(
                    // unchecked - is safe here, overflow is not possible due to above if conditional
                    forRedemption - oracleData.externalUnderlyingAvailableForWithdraw
                );
            }
        }
```

When `targetUtilization==0`, it returns `targetAmount ==0`.
It lacks the other judgments of whether the current `externalUnderlyingAvailableForWithdraw` is sufficient.
Exceeding `externalUnderlyingAvailableForWithdraw` may cause `_rebalanceCurrency()` to revert.

For example:
`currentExternalUnderlyingLend = 100`
`externalUnderlyingAvailableForWithdraw = 99`
If `targetUtilization` is modified to `0`,
then `targetAmount` should be `1`, not `0`.
`0` will cause an error due to insufficient available balance for withdrawal.

So, it should still try to withdraw as much deposit as possible first, wait for replenishment, and then withdraw the remaining deposit until the deposit is cleared.

## Impact

A too small `targetAmount` may cause AAVE withdraw to fail, thereby causing the inability to `setRebalancingTargets()` to fail.

## Code Snippet
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/ExternalLending.sol#L44
## Tool used

Manual Review

## Recommendation

Remove `targetUtilization == 0` directly returning 0.

The subsequent logic of the method can handle `targetUtilization == 0` normally and will not cause an error.

```diff
    function getTargetExternalLendingAmount(
        Token memory underlyingToken,
        PrimeCashFactors memory factors,
        RebalancingTargetData memory rebalancingTargetData,
        OracleData memory oracleData,
        PrimeRate memory pr
    ) internal pure returns (uint256 targetAmount) {
        // Short circuit a zero target
-       if (rebalancingTargetData.targetUtilization == 0) return 0;
```