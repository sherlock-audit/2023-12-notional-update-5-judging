Short Candy Salmon

medium

# _isExternalLendingUnhealthy() using stale factors

## Summary
In `checkRebalance() -> _isExternalLendingUnhealthy() -> getTargetExternalLendingAmount(factors)`
using stale `factors` will lead to inaccurate `targetAmount`, which in turn will cause `checkRebalance()` that should have been rebalance to not execute.

## Vulnerability Detail

rebalancingBot uses `checkRebalance()` to return the `currencyIds []` that need to be `rebalance`.

call order : 
`checkRebalance()` -> `_isExternalLendingUnhealthy()` -> `ExternalLending.getTargetExternalLendingAmount(factors)`

```solidity
    function _isExternalLendingUnhealthy(
        uint16 currencyId,
        IPrimeCashHoldingsOracle oracle,
        PrimeRate memory pr
    ) internal view returns (bool isExternalLendingUnhealthy, OracleData memory oracleData, uint256 targetAmount) {
...

@>      PrimeCashFactors memory factors = PrimeCashExchangeRate.getPrimeCashFactors(currencyId);
        Token memory underlyingToken = TokenHandler.getUnderlyingToken(currencyId);

        targetAmount = ExternalLending.getTargetExternalLendingAmount(
            underlyingToken, factors, rebalancingTargetData, oracleData, pr
        );
```

A very important logic is to get `targetAmount`. 
The calculation of this value depends on `factors`. 
But currently used is `PrimeCashFactors memory factors = PrimeCashExchangeRate.getPrimeCashFactors(currencyId);`. 
This is not the latest. It has not been aggregated yet. 
The correct one should be `( /* */,factors) = PrimeCashExchangeRate.getPrimeCashRateView();`.

## Impact

Due to the incorrect `targetAmount`, it may cause the `currencyId` that should have been re-executed `Rebalance` to not execute `rebalance`, increasing the risk of the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L414

## Tool used

Manual Review

## Recommendation

```diff
    function _isExternalLendingUnhealthy(
        uint16 currencyId,
        IPrimeCashHoldingsOracle oracle,
        PrimeRate memory pr
    ) internal view returns (bool isExternalLendingUnhealthy, OracleData memory oracleData, uint256 targetAmount) {
...

-       PrimeCashFactors memory factors = PrimeCashExchangeRate.getPrimeCashFactors(currencyId);
+       ( /* */,PrimeCashFactors memory factors) = PrimeCashExchangeRate.getPrimeCashRateView();
        Token memory underlyingToken = TokenHandler.getUnderlyingToken(currencyId);

        targetAmount = ExternalLending.getTargetExternalLendingAmount(
            underlyingToken, factors, rebalancingTargetData, oracleData, pr
        );
```
