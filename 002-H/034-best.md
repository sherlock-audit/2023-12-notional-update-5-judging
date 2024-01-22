Strong Leather Toad

high

# `_previewDeposit` function returns max fCash when an error occurs

## Summary

The `_previewDeposit` function will return the max fCash when an error has occurred. This means that when an error has occurred, for any amount of assets that the user deposited (e.g., 1 wei), the function will return the maximum amount of fCash, which is incorrect and dangerous. By chaining up this specific issue with how the minting works internally in Notional when an account holds a cash balance, one could exploit these to drain the funds from the wrapper's account.

## Vulnerability Detail

When there is an error when executing the `NotionalV2.getfCashLendFromDeposit` function, the `shares` will always be set to `maxFCash` as shown in Line 105 below:

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L105

```solidity
File: wfCashERC4626.sol
089:     function _previewDeposit(uint256 assets) internal view returns (uint256 shares, uint256 maxFCash) {
090:         if (hasMatured()) return (0, 0); 
091:         // This is how much fCash received from depositing assets
092:         (uint16 currencyId, uint40 maturity) = getDecodedID(); 
093:         (/* */, maxFCash) = getTotalFCashAvailable();
094: 
095:         try NotionalV2.getfCashLendFromDeposit(
096:             currencyId,
097:             assets,
098:             maturity,
099:             0,
100:             block.timestamp,
101:             true
102:         ) returns (uint88 s, uint8, bytes32) { 
103:             shares = s;
104:         } catch {
105:             shares = maxFCash; 
106:         }
107:     }
```

Assume that a malicious user called Bob calls the following `deposit` function with `assets` set to `1 wei`. An error occurs. The `_previewDeposit` function at Line 190 will return the `maxFCash` (1,000,000 fUSDC) and set the `shares` variable to 1,000,000 fUSDC.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L191

```solidity
File: wfCashERC4626.sol
189:     function deposit(uint256 assets, address receiver) external override returns (uint256) {
190:         (uint256 shares, uint256 maxFCash) = _previewDeposit(assets);
191:         // Will revert if matured
192:         _mintInternal(assets, _safeUint88(shares), receiver, 0, maxFCash); 
193:         emit Deposit(msg.sender, receiver, assets, shares);
194:         return shares;
195:     }
```

When the `_mintInternal` function is executed, the `depositAmountExternal` function will be set to 1 wei, while `fCashAmount` is set to 1,000,000 fUSDC. Assume that the wrapper account holds a large cash balance. Thus, the `_lendLegacy` function at Line 69 will be executed. Within the `_lendLegacy` function, the `BatchAction._batchBalanceAndTradeAction` function will be called internally.

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

Subsequently, the `_executeTrades` function at Line 289 will be executed to lend 1,000,000 fUSDC, and `netCash` will be set to a large negative value. At Line 298 below, the account's `balanceState.netCashChange` will incur a large negative balance.

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

As a result, Bob minted 1,000,000 wfUSDC with 1 wei of deposit.

## Impact

Loss of assets as the cash of the wrapper's account will be lost.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L105

## Tool used

Manual Review

## Recommendation

Consider reverting if the `_previewDeposit` function reverts. Returning a max or zero value when an error occurs within the `_previewDeposit` function would be incorrect under any circumstance.