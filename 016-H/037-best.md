Strong Leather Toad

high

# Malicious users can mint wfCash by using the wrapper's cash balance

## Summary

Malicious users make use of the cash balance on the wrapper account to mint wfCash, resulting in a loss of assets for the wrapper account.

## Vulnerability Detail

Bob (malicious user) calls the `mintViaUnderlying` function with `depositAmountExternal` set to 1 wei, and `fCashAmount` set to 1,000 fUSDC.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L25

```solidity
File: wfCashLogic.sol
20:     /// @notice Lends deposit amount in return for fCashAmount using underlying tokens
21:     /// @param depositAmountExternal amount of cash to deposit into this method
22:     /// @param fCashAmount amount of fCash to purchase (lend)
23:     /// @param receiver address to receive the fCash shares
24:     /// @param minImpliedRate minimum annualized interest rate to lend at
25:     function mintViaUnderlying(
26:         uint256 depositAmountExternal,
27:         uint88 fCashAmount,
28:         address receiver,
29:         uint32 minImpliedRate
30:     ) external override {
31:         (/* */, uint256 maxFCash) = getTotalFCashAvailable();
32:         _mintInternal(depositAmountExternal, fCashAmount, receiver, minImpliedRate, maxFCash);
33:     }
```

When the `_mintInternal` function is executed, the `depositAmountExternal` function will be set to 1 wei, while `fCashAmount` is set to 1,000 fUSDC. Assume that the wrapper account holds a large cash balance. Thus, the `_lendLegacy` at Line 69 will be executed. Within the `_lendLegacy` function, the `BatchAction._batchBalanceAndTradeAction` function will be called internally.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L69

```solidity
File: wfCashLogic.sol
35:     function _mintInternal(
36:         uint256 depositAmountExternal, 
37:         uint88 fCashAmount, 
38:         address receiver,
39:         uint32 minImpliedRate,
40:         uint256 maxFCash
41:     ) internal nonReentrant {
..SNIP..
68:         } else if (isETH || hasTransferFee || getCashBalance() > 0) { 
69:             _lendLegacy(currencyId, depositAmountExternal, fCashAmount, minImpliedRate, msgValue, isETH);
```

Within the `BatchAction._batchBalanceAndTradeAction` function below, it will first call the `_executeDepositAction` function to deposit 1 wei of USDC, which does not materially increase the cash balance of the wrapper account. The `balanceState.netCashChange` to be close to zero after the deposit at this point.

Subsequently, the `_executeTrades` function at Line 289 will be executed to lend 1,000 fUSDC, and `netCash` will be set to a large negative value. At Line 298 below, the account's `balanceState.netCashChange` will incur a large negative balance.

Next, the `_calculateWithdrawActionAndFinalize` function at Line 301 will be executed, which will internally call the `finalizeWithWithdraw` function and `_finalize` function.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/BatchAction.sol#L255

```solidity
File: BatchAction.sol
255:     function _batchBalanceAndTradeAction(
..SNIP..
279:             // Does not revert on invalid action types here, they also have no effect.
280:             _executeDepositAction(
281:                 account,
282:                 balanceState,
283:                 action.actionType,
284:                 action.depositActionAmount
285:             ); 
286: 
287:             if (action.trades.length > 0) {
288:                 int256 netCash;
289:                 (netCash, portfolioState) = _executeTrades( 
290:                     account,
291:                     action.currencyId,
292:                     action.trades,
293:                     accountContext,
294:                     portfolioState
295:                 );
296: 
297:                 // If the account owes cash after trading, ensure that it has enough
298:                 balanceState.netCashChange = balanceState.netCashChange.add(netCash); 
299:             }
300: 
301:             _calculateWithdrawActionAndFinalize(
302:                 account,
303:                 accountContext,
304:                 balanceState,
305:                 action.withdrawAmountInternalPrecision,
306:                 action.withdrawEntireCashBalance,
307:                 action.redeemToUnderlying
308:             );
309:         }
```

At Line 175 below, `balanceState.primeCashWithdraw` is zero, and `balanceState.netCashChange` is a large negative value. Thus, the `totalCashChange` will be a large negative value.

Since the wrapper has a large cash balance (`balanceState.storedCashBalance`), after deducting the `totalCashChange`, there is still a significantly large amount of cash balance left, and the free collateral check will pass.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/BalanceHandler.sol#L175

```solidity
File: BalanceHandler.sol
151:     function  _finalize(
..SNIP..
174:             // No changes to total cash after this point
175:             int256 totalCashChange = balanceState.netCashChange.add(balanceState.primeCashWithdraw);
..SNIP..
200:             if (totalCashChange != 0) {
201:                 balanceState.storedCashBalance = balanceState.storedCashBalance.add(totalCashChange);
202:                 mustUpdate = true;
203:             }
```

As a result, Bob minted 1,000 wfUSDC with 1 wei of deposit.

## Impact

Loss of assets as the cash of the wrapper's account will be lost.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L25

## Tool used

Manual Review

## Recommendation

For the `mintViaUnderlying` function, consider removing the ability to control the amount of fCash to lent. The amount of fCash to lent should be computed based on the amount of deposit.

```diff
    /// @notice Lends deposit amount in return for fCashAmount using underlying tokens
    /// @param depositAmountExternal amount of cash to deposit into this method
-    /// @param fCashAmount amount of fCash to purchase (lend)
    /// @param receiver address to receive the fCash shares
    /// @param minImpliedRate minimum annualized interest rate to lend at
    function mintViaUnderlying(
        uint256 depositAmountExternal,
-        uint88 fCashAmount,
        address receiver,
        uint32 minImpliedRate
    ) external override {
        (/* */, uint256 maxFCash) = getTotalFCashAvailable();
+        uint88 = fCashAmount = /** Compute amount of fCash to lend based on the deposited amount **/
        _mintInternal(depositAmountExternal, fCashAmount, receiver, minImpliedRate, maxFCash);
    }
```