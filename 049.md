Short Candy Salmon

medium

# getOracleData() maxExternalDeposit not accurate

## Summary
in `getOracleData() ` 
The calculation of `maxExternalDeposit` lacks consideration for `reserve.accruedToTreasury`.
This leads to `maxExternalDeposit` being too large, causing `Treasury.rebalance()` to fail.


## Vulnerability Detail
in `getOracleData()`

```solidity
    function getOracleData() external view override returns (OracleData memory oracleData) {
...
        (/* */, uint256 supplyCap) = IPoolDataProvider(POOL_DATA_PROVIDER).getReserveCaps(underlying);
        // Supply caps are returned as whole token values
        supplyCap = supplyCap * UNDERLYING_PRECISION;
        uint256 aTokenSupply = IPoolDataProvider(POOL_DATA_PROVIDER).getATokenTotalSupply(underlying);

        // If supply cap is zero, that means there is no cap on the pool
        if (supplyCap == 0) {
            oracleData.maxExternalDeposit = type(uint256).max;
        } else if (supplyCap <= aTokenSupply) {
            oracleData.maxExternalDeposit = 0;
        } else {
            // underflow checked as consequence of if / else statement
@>          oracleData.maxExternalDeposit = supplyCap - aTokenSupply;
        }
```

However, AAVE's restrictions are as follows:
[ValidationLogic.sol#L81-L88](https://github.com/aave/aave-v3-core/blob/6070e82d962d9b12835c88e68210d0e63f08d035/contracts/protocol/libraries/logic/ValidationLogic.sol#L81-L88)
```solidity
    require(
      supplyCap == 0 ||
        ((IAToken(reserveCache.aTokenAddress).scaledTotalSupply() +
          uint256(reserve.accruedToTreasury)).rayMul(reserveCache.nextLiquidityIndex) + amount) <=
        supplyCap * (10 ** reserveCache.reserveConfiguration.getDecimals()),
      Errors.SUPPLY_CAP_EXCEEDED
    );
  }
```

The current implementation lacks subtraction of `uint256(reserve.accruedToTreasury)).rayMul(reserveCache.nextLiquidityIndex)`.


## Impact

An overly large `maxExternalDeposit` may cause `rebalance()` to be unable to execute.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/pCash/AaveV3HoldingsOracle.sol#L160

## Tool used

Manual Review

## Recommendation

 subtract `uint256(reserve.accruedToTreasury)).rayMul(reserveCache.nextLiquidityIndex) `