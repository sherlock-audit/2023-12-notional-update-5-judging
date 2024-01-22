Strong Leather Toad

high

# Residual ETH not sent back when `batchBalanceAndTradeAction` executed

## Summary

Residual ETH was not sent back when `batchBalanceAndTradeAction` function was executed, resulting in a loss of assets.

## Vulnerability Detail

Per the comment at Line 122 below, when there is residual ETH, native ETH will be sent from Notional V3 to the wrapper contract. In addition, per the comment at Line 109, it is often the case to have an excess amount to be refunded to the users.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L122

```solidity
File: wfCashLogic.sol
094:     function _lendLegacy(
File: wfCashLogic.sol
109:         // If deposit amount external is in excess of the cost to purchase fCash amount (often the case),
110:         // then we need to return the difference between postTradeCash - preTradeCash. This is done because
111:         // the encoded trade does not automatically withdraw the entire cash balance in case the wrapper
112:         // is holding a cash balance.
113:         uint256 preTradeCash = getCashBalance();
114: 
115:         BalanceActionWithTrades[] memory action = EncodeDecode.encodeLegacyLendTrade(
116:             currencyId,
117:             getMarketIndex(),
118:             depositAmountExternal,
119:             fCashAmount,
120:             minImpliedRate
121:         );
122:         // Notional will return any residual ETH as the native token. When we _sendTokensToReceiver those
123:         // native ETH tokens will be wrapped back to WETH.
124:         NotionalV2.batchBalanceAndTradeAction{value: msgValue}(address(this), action); 
125: 
126:         uint256 postTradeCash = getCashBalance(); 
127: 
128:         if (preTradeCash != postTradeCash) { 
129:             // If ETH, then redeem to WETH (redeemToUnderlying == false)
130:             NotionalV2.withdraw(currencyId, _safeUint88(postTradeCash - preTradeCash), !isETH);
131:         }
132:     }
```

This is due to how the `depositUnderlyingExternal` function within Notional V3 is implemented. The `batchBalanceAndTradeAction` will trigger the `depositUnderlyingExternal` function. Within the `depositUnderlyingExternal` function at Line 196, excess ETH will be transferred back to the account (wrapper address) in Native ETH term. 

Note that for other ERC20 tokens, such as DAI or USDC, the excess will be added to the wrapper's cash balance, and this issue will not occur.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/TokenHandler.sol#L196

```solidity
File: TokenHandler.sol
181:     function depositUnderlyingExternal(
182:         address account,
183:         uint16 currencyId,
184:         int256 _underlyingExternalDeposit,
185:         PrimeRate memory primeRate,
186:         bool returnNativeTokenWrapped
187:     ) internal returns (int256 actualTransferExternal, int256 netPrimeSupplyChange) {
188:         uint256 underlyingExternalDeposit = _underlyingExternalDeposit.toUint();
189:         if (underlyingExternalDeposit == 0) return (0, 0);
190: 
191:         Token memory underlying = getUnderlyingToken(currencyId);
192:         if (underlying.tokenType == TokenType.Ether) {
193:             // Underflow checked above
194:             if (underlyingExternalDeposit < msg.value) {
195:                 // Transfer any excess ETH back to the account
196:                 GenericToken.transferNativeTokenOut(
197:                     account, msg.value - underlyingExternalDeposit, returnNativeTokenWrapped
198:                 );
199:             } else {
200:                 require(underlyingExternalDeposit == msg.value, "ETH Balance");
201:             }
202: 
203:             actualTransferExternal = _underlyingExternalDeposit;
```

In the comment, it mentioned that any residual ETH in native token will be wrapped back to WETH by the `_sendTokensToReceiver`.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L122

```solidity
File: wfCashLogic.sol
094:     function _lendLegacy(
..SNIP..
122:         // Notional will return any residual ETH as the native token. When we _sendTokensToReceiver those
123:         // native ETH tokens will be wrapped back to WETH.
```

However, the current implementation of the `_sendTokensToReceiver`, as shown below, does not wrap the Native ETH to WETH. Thus, the residual ETH will not be sent back to the users and stuck in the contract.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L331

```solidity
File: wfCashLogic.sol
331:     function _sendTokensToReceiver( 
332:         IERC20 token,
333:         address receiver,
334:         bool isETH,
335:         uint256 balanceBefore
336:     ) private returns (uint256 tokensTransferred) {
337:         uint256 balanceAfter = isETH ? WETH.balanceOf(address(this)) : token.balanceOf(address(this)); 
338:         tokensTransferred = balanceAfter - balanceBefore; 
339: 
340:         if (isETH) {
341:             // No need to use safeTransfer for WETH since it is known to be compatible
342:             IERC20(address(WETH)).transfer(receiver, tokensTransferred); 
343:         } else if (tokensTransferred > 0) { 
344:             token.safeTransfer(receiver, tokensTransferred); 
345:         }
346:     }
```

## Impact

Loss of assets as the residual ETH is not sent to the users.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L122

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/TokenHandler.sol#L196

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L331

## Tool used

Manual Review

## Recommendation

If the underlying is ETH, measure the Native ETH balance before and after the `batchBalanceAndTradeAction` is executed. Forward any residual Native ETH to the users, if any.