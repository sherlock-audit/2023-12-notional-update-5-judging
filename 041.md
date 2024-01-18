Strong Leather Toad

high

# `maxImpliedRate` is hardcoded to zero

## Summary

The slippage control (`maxImpliedRate`) was found to be hardcoded to zero. As a result, the trade will proceed even if the user suffers huge slippage, resulting in the user receiving less cash than expected (loss of cash).

## Vulnerability Detail

The `wfCashERC4626.withdraw` and `wfCashERC4626.redeem` relies on the `_redeemInternal` function to redeem the shares (wfCash). However, it was found that the `RedeemOpts.maxImpliedRate` is hardcoded to zero.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L262

```solidity
File: wfCashERC4626.sol
250:     function _redeemInternal(
251:         uint256 shares,
252:         address receiver,
253:         address owner
254:     ) private {
255:         _burnInternal(
256:             owner,
257:             shares,
258:             RedeemOpts({
259:                 redeemToUnderlying: true,
260:                 transferfCash: false,
261:                 receiver: receiver,
262:                 maxImpliedRate: 0
263:             })
264:         );
265:     }
```

Assume that the fCash has not matured yet and the wrapper account does not have any cash balance. As such, the `wfCashLogic._sellfCash` function will be executed, which in turn calls the `NotionalV2.batchBalanceAndTradeAction` function. The `maxImpliedRate`, which is set to zero, is used to set the `rateLimit` of the trade.

```solidity
File: wfCashLogic.sol
274:     /// @dev Sells an fCash share back on the Notional AMM
275:     function _sellfCash(
276:         address receiver,
277:         uint256 fCashToSell,
278:         uint32 maxImpliedRate
279:     ) private returns (uint256 tokensTransferred) {
..SNIP..
307:         if (fCashToSell > 0) {
308:             // Sells fCash on Notional AMM (via borrowing)
309:             BalanceActionWithTrades[] memory action = EncodeDecode.encodeBorrowTrade(
310:                 currencyId,
311:                 getMarketIndex(),
312:                 _safeUint88(fCashToSell),
313:                 maxImpliedRate
314:             ); 
315:             NotionalV2.batchBalanceAndTradeAction(address(this), action); 
316:         }
```

After the trade is executed, there is an option to check the slippage if the `rateLimit` is not set to zero. This is to ensure that the trade does not incur excessive slippage beyond the user's acceptable limit. Since the `rateLimit` has been hardcoded to zero, the slippage control is effectively disabled.

As a result, the trade will proceed even if the user suffers huge slippage, resulting in the user receiving less cash than expected (loss of cash).

```solidity
File: TradingAction.sol
207:     function _executeLendBorrowTrade(
..SNIP..
229:         uint256 postFeeInterestRate;
230:         (cashAmount, postFeeInterestRate) = market.executeTrade( // @audit-info cashAmount => positve (when borrow) OR negative (when lend)
231:             account,
232:             cashGroup,
233:             fCashAmount,
234:             market.maturity.sub(blockTime),
235:             marketIndex
236:         );
237: 
238:         uint256 rateLimit = uint256(uint32(bytes4(trade << 104)));
239:         if (rateLimit != 0) {
240:             if (tradeType == TradeActionType.Borrow) {
241:                 // Do not allow borrows over the rate limit
242:                 require(postFeeInterestRate <= rateLimit, "Trade failed, slippage");
243:             } else {
244:                 // Do not allow lends under the rate limit
245:                 require(postFeeInterestRate >= rateLimit, "Trade failed, slippage");
246:             }
247:         }
```

## Impact

The trade will proceed even if the user suffers huge slippage, resulting in the user receiving less cash than expected (loss of cash).

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L262

## Tool used

Manual Review

## Recommendation

Consider updating the `redeem` and `withdraw` functions to allow users to define the rate limit. If the changes are not possible due to the requirement to align with ERC4626 interfaces, consider documenting the risk that it is possible for the users to incur slippage during the trade when these functions are used.