Strong Leather Toad

high

# Unable to limit the loss when redeeming wfCash before maturity

## Summary

Users are unable to limit the loss when redeeming wfCash before maturity, leading to a loss of assets as they might receive less prime cash than expected in return.

## Vulnerability Detail

When the users redeem its wfCash prior to maturity, they will be sold back on the Notional AMM in exchange for the prime cash. For simplicity's sake, assume that there is no more fCash on the wrapper to be sold to the AMM. In this case, it will compute the amount of prime cash to be withdrawn and sent to the users in Line 293 below:

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L293

```solidity
File: wfCashLogic.sol
274:     /// @dev Sells an fCash share back on the Notional AMM
275:     function _sellfCash(
276:         address receiver,
277:         uint256 fCashToSell,
278:         uint32 maxImpliedRate
279:     ) private returns (uint256 tokensTransferred) {
280:         (IERC20 token, bool isETH) = getToken(true); 
281:         uint256 balanceBefore = isETH ? WETH.balanceOf(address(this)) : token.balanceOf(address(this)); 
282:         uint16 currencyId = getCurrencyId(); 
283: 
284:         (uint256 initialCashBalance, uint256 fCashBalance) = getBalances(); 
285:         bool hasInsufficientfCash = fCashBalance < fCashToSell; 
286: 
287:         uint256 primeCashToWithdraw; 
288:         if (hasInsufficientfCash) {
289:             // If there is insufficient fCash, calculate how much prime cash would be purchased if the
290:             // given fCash amount would be sold and that will be how much the wrapper will withdraw and
291:             // send to the receiver. Since fCash always sells at a discount to underlying prior to maturity,
292:             // the wrapper is guaranteed to have sufficient cash to send to the account.
293:             (/* */, primeCashToWithdraw, /* */, /* */) = NotionalV2.getPrincipalFromfCashBorrow( 
294:                 currencyId,
295:                 fCashToSell, 
296:                 getMaturity(),
297:                 0, 
298:                 block.timestamp
299:             ); 
300:             // If this is zero then it signifies that the trade will fail.
301:             require(primeCashToWithdraw > 0, "Redeem Failed"); 
302: 
303:             // Re-write the fCash to sell to the entire fCash balance.
304:             fCashToSell = fCashBalance;
305:         }
```

At Line 297 above, it was observed that `maxBorrowRate` was set to zero when the `getPrincipalFromfCashBorrow` function was executed, which effectively disables the slippage protection.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/CalculationViews.sol#L440

```solidity
File: CalculationViews.sol
429:     /// @notice Returns the amount of underlying cash and asset cash required to borrow fCash. When specifying a
430:     /// trade, choose to receive either underlying or asset tokens (not both). Asset tokens tend to be more gas efficient.
431:     /// @param currencyId id number of the currency
432:     /// @param fCashBorrow amount of fCash (in underlying) that will be received at maturity. Always 8 decimal precision.
433:     /// @param maturity the maturity of the fCash to lend
434:     /// @param maxBorrowRate the maximum borrow rate (slippage protection)
435:     /// @param blockTime the block time for when the trade will be calculated
436:     /// @return borrowAmountUnderlying the amount of underlying tokens the borrower will receive
437:     /// @return borrowAmountPrimeCash the amount of prime cash tokens the borrower will receive
438:     /// @return marketIndex the corresponding market index for the lending
439:     /// @return encodedTrade the encoded bytes32 object to pass to batch trade
440:     function getPrincipalFromfCashBorrow(
441:         uint16 currencyId,
442:         uint256 fCashBorrow,
443:         uint256 maturity,
444:         uint32 maxBorrowRate,
445:         uint256 blockTime
446:     ) external view override returns (
```

Within the `CalculationViews.getPrincipalFromfCashBorrow` view function, it will rely on the `InterestRateCurve.calculatefCashTrade` function to compute the cash to be returned based on the current interest rate model.

The Notional AMM uses a two-kink interest rate model, and the interest rate is computed based on the utilization rate (https://docs.notional.finance/notional-v3/prime-money-market/interest-rate-model) as shown below. The interest rate is used to discount the fCash back to cash before maturity. In summary, a higher interest rate will cause the fCash to be discounted more, and the current cash value will be smaller.

Note that after Kink 2, there will be a sharp increase in the interest rate. Assume the current utilization rate is close to Kink 2. Bob decided to sell its wfCash before maturity. Before the actual transaction, Bob simulated the transaction and was happy with the computed cash amount as the interest rate still falls within Kink 1 and Kink 2, where the slope is gentle. 

However, when Bob's transaction is executed, it ends up trading with a high interest rate, and the `CalculationViews.getPrincipalFromfCashBorrow` function returns a smaller value, resulting in less prime cash than expected being withdrawn to Bob. This is because the utilization rate has increased beyond Kink 2 due to trading activities, which might be malicious or benign, before his transaction is executed within the same block.

![IR](https://github.com/sherlock-audit/2023-12-notional-update-5-xiaoming9090/assets/102820284/a44767a1-6371-4bf9-8911-818d69c76e6e)

## Impact

Loss of assets as users might receive less prime cash than expected in return when redeeming before maturity.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L293

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/CalculationViews.sol#L440

## Tool used

Manual Review

## Recommendation

Update the `_sellfCash` function to allow the callers to define the `maxBorrowRate` of the `getPrincipalFromfCashBorrow` function to limit the loss.