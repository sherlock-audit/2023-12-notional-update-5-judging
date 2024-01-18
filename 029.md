Strong Leather Toad

high

# Accumulated or earned pCash stuck in the wrapper contract

## Summary

Accumulated or earned pCash is stuck in the wrapper contract as there is no way to withdraw them.

## Vulnerability Detail

Assume that the `maxFCash` cap has been reached. Bob intended to mint 1000 wfDAI; thus, he will be minting at zero interest. Bob deposited 1000 worth of DAI. 

1000 DAI will be deposited to Notional via the `Notional.depositUnderlyingToken` function at Line 67 below, and $x$ pDAI is minted to the wrapper address. Subsequently, the wrapper minted 1000 wfDAI to Bob's address.

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
```

Assume that Bob intended to redeem his 1000 wfDAI prior to maturity. When the fCash has not matured yet, the fCash amount will be discounted back to obtain its present value. Due to the discounting, Bob will receive less than 1000 DAI back, and the amount of pDAI (`primeCashToWithdraw`) to be withdrawn from the wrapper account will be less than $x$ pDAI. Refer to the comment below for more details.

Let the `primeCashToWithdraw` be $y$ pDAI. In this case, $y\ pDAI < x\ pDAI$.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L289

```solidity
File: wfCashLogic.sol
275:     function _sellfCash(
..SNIP..
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
..SNIP..
```

After the transaction, the wrapper will effectively accumulate or gain $(x - y) \ pDAI$.

When a large amount of minting occurs at zero interest rate, and the early redemptions prior to maturity happen frequently, it will result in a significant amount of pCash accumulated or gained in the wrapper account.

However, there is no function within the wrapper contract that allows the protocol to retrieve the accumulated or gained pCash that resides on the wrapper account after maturity. Thus, this pCash is effectively stuck in the contract.

## Impact

Unable to retrieve the gained/accumulated cash. Loss of assets as cash is stuck in the contract.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L67

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L289

## Tool used

Manual Review

## Recommendation

Consider implementing an additional function to forward gained/accumulated cash in the wrapper contract to the Notional Treasury after maturity.