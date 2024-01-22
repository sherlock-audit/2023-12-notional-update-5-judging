Strong Leather Toad

high

# Residual ETH will not be sent back to users during the minting of wfCash

## Summary

Residual ETH will not be sent back to users, resulting in a loss of assets.

## Vulnerability Detail

At Line 67, residual ETH within the `depositUnderlyingToken` function will be sent as Native ETH back to the `msg.sender`, which is this wfCash Wrapper contract. 

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L67

```solidity
File: wfCashLogic.sol
35:     function _mintInternal(
..SNIP..
60:         if (maxFCash < fCashAmount) {
61:             // NOTE: lending at zero
62:             uint256 fCashAmountExternal = fCashAmount * precision / uint256(Constants.INTERNAL_TOKEN_PRECISION); 
63:             require(fCashAmountExternal <= depositAmountExternal); 
64: 
65:             // NOTE: Residual (depositAmountExternal - fCashAmountExternal) will be transferred
66:             // back to the account
67:             NotionalV2.depositUnderlyingToken{value: msgValue}(address(this), currencyId, fCashAmountExternal);
..SNIP..
87:         // Residual tokens will be sent back to msg.sender, not the receiver. The msg.sender
88:         // was used to transfer tokens in and these are any residual tokens left that were not
89:         // lent out. Sending these tokens back to the receiver risks them getting locked on a
90:         // contract that does not have the capability to transfer them off
91:         _sendTokensToReceiver(token, msg.sender, isETH, balanceBefore);
```

Within the `depositUnderlyingToken` function Line 108 below, the `returnExcessWrapped` parameter is set to `false`, which means it will not wrap the residual ETH, and that Native ETH will be sent back to the caller (wrapper contract)

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/AccountAction.sol#L108

```solidity
File: AccountAction.sol
89:     function depositUnderlyingToken(
90:         address account,
91:         uint16 currencyId,
92:         uint256 amountExternalPrecision
93:     ) external payable nonReentrant returns (uint256) {
..SNIP..
File: AccountAction.sol
105:         int256 primeCashReceived = balanceState.depositUnderlyingToken(
106:             msg.sender,
107:             SafeInt256.toInt(amountExternalPrecision),
108:             false // there should never be excess ETH here by definition
109:         );
```

balanceBefore = amount of WETH before the deposit, balanceAfter = amount of WETH after the deposit. 

When the `_sendTokensToReceiver` is executed, these two values are going to be the same since it is Native ETH that is sent to the wrapper instead of WETH. As a result, the Native ETH that the wrapper received is not forwarded to the users.

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

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L67

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/actions/AccountAction.sol#L108

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L331

## Tool used

Manual Review

## Recommendation

If the underlying is ETH, measure the Native ETH balance before and after the `depositUnderlyingToken` is executed. Forward any residual Native ETH to the users, if any.