Sweet Blood Cod

high

# `ExternalLending` - executing redemptions for fee-on-transfer tokens from AaveV3 will always revert

## Summary
When the Treasury rebalances and has to redeem aTokens from AaveV3, it checks that the actual amount withdrawn is greater than or equal to the set `withdrawAmount`. This check will always fail for fee-on-transfer tokens since the `withdrawAmount` does not account for the transfer fee.

## Vulnerability Detail
When the Treasury rebalances and has to redeem aWETH from AaveV3 it executes calls that were encoded in `AaveV3HoldingsOracle._getRedemptionCalldata()`:
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/pCash/AaveV3HoldingsOracle.sol#L61-L81
```solidity
        address[] memory targets = new address[](UNDERLYING_IS_ETH ? 2 : 1);
        bytes[] memory callData = new bytes[](UNDERLYING_IS_ETH ? 2 : 1);
        targets[0] = LENDING_POOL;
        callData[0] = abi.encodeWithSelector(
            ILendingPool.withdraw.selector, underlyingToken, withdrawAmount, address(NOTIONAL)
        );

        if (UNDERLYING_IS_ETH) {
            // Aave V3 returns WETH instead of native ETH so we have to unwrap it here
            targets[1] = address(Deployments.WETH);
            callData[1] = abi.encodeWithSelector(WETH9.withdraw.selector, withdrawAmount);
        }

        data = new RedeemData[](1);
        // Tokens with less than or equal to 8 decimals sometimes have off by 1 issues when depositing
        // into Aave V3. Aave returns one unit less than has been deposited. This adjustment is applied
        // to ensure that this unit of token is credited back to prime cash holders appropriately.
        uint8 rebasingTokenBalanceAdjustment = UNDERLYING_DECIMALS <= 8 ? 1 : 0;
        data[0] = RedeemData(
            targets, callData, withdrawAmount, ASSET_TOKEN, rebasingTokenBalanceAdjustment
        );
```

Note that the third field in the `RedeemData` struct is the `expectedUnderlying` field which is set to the `withdrawAmount` and that `withdrawAmount` is a value greater than zero. 

Execution of redemption is done in `ExternalLending.executeMoneyMarketRedemptions()`.
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/ExternalLending.sol#L163-L172
```solidity
            for (uint256 j; j < data.targets.length; j++) {
                GenericToken.executeLowLevelCall(data.targets[j], 0, data.callData[j]);
            }

            // Ensure that we get sufficient underlying on every redemption
            uint256 newUnderlyingBalance = TokenHandler.balanceOf(underlyingToken, address(this));
            uint256 underlyingBalanceChange = newUnderlyingBalance.sub(oldUnderlyingBalance);
            // If the call is not the final redemption, then expectedUnderlying should
            // be set to zero.
            require(data.expectedUnderlying <= underlyingBalanceChange);
```

The `require` statement expects that the change in token balance is always greater than or equal to `data.expectedUnderlying`. However, `data.expectedUnderlying` was calculated with:
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L452
```solidity
redeemAmounts[0] = currentAmount - targetAmount;
```

It does not account for transfer fees. In effect, that check will always revert when the underlying being withdrawn is a fee-on-transfer token.

## Impact
Rebalancing will always fail when redemption of a fee-on-transfer token is executed. This will also break withdrawals of prime cash from Notional of fee-on-transfer tokens that require redemption from AaveV3.
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/TokenHandler.sol#L243-L244
```solidity
    uint256 withdrawAmount = uint256(netTransferExternal.neg());
    ExternalLending.redeemMoneyMarketIfRequired(currencyId, underlying, withdrawAmount);
```
This means that these tokens can only be deposited into AaveV3 but can never redeemed. This can lead to insolvency of the protocol.

## Code Snippet
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/pCash/AaveV3HoldingsOracle.sol#L61-L81
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/TreasuryAction.sol#L452
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/ExternalLending.sol#L163-L172
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/TokenHandler.sol#L243-L244

## Tool used
Manual Review

## Recommendation
When computing for the `withdrawAmount / data.expectedUnderlying`, it should account for the transfer fees. The pseudocode for the computation may look like so:
```pseudocode
withdrawAmount = currentAmount - targetAmount
if (underlyingToken.hasTransferFee) {
  withdrawAmount = withdrawAmount / (100% - underlyingToken.transferFeePercent)
}
```
