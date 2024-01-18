Strong Leather Toad

high

# Cash in wrapper account can be drained due to rounding error

## Summary

Cash in the wrapper account can be drained due to rounding errors when the underlying decimals are less than Notional's internal token precision (1e8).

## Vulnerability Detail

Assume that the underlying token of the wrapper is USDC with 6 decimal precision. The `Constants.INTERNAL_TOKEN_PRECISION)` is 1e8.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/lib/Constants.sol#L11

```solidity
File: Constants.sol
10:     int256 internal constant INTERNAL_TOKEN_PRECISION = 1e8;
```

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L114

```solidity
File: wfCashERC4626.sol
114:     function _previewMint(uint256 shares) internal view returns (uint256 assets, uint256 maxFCash) {
115:         if (hasMatured()) return (0, 0); 
116: 
117:         // This is how much fCash received from depositing assets
118:         (uint16 currencyId, uint40 maturity) = getDecodedID(); 
119:         (/* */, maxFCash) = getTotalFCashAvailable(); 
120:         if (maxFCash < shares) {
121:             (/* */, int256 precision) = getUnderlyingToken(); 
122:             require(precision > 0); 
123:             // Lending at zero interest means that 1 fCash unit is equivalent to 1 asset unit
124:             assets = shares * uint256(precision) / uint256(Constants.INTERNAL_TOKEN_PRECISION); 
```

Based on the above information, the formula at Line 124 above for computing the number of assets is as follows:

```solidity
assets = shares * 1e6 / 1e8
```

For `shares` less than or equal to 99, the assets will round down to zero due to a Solidity rounding error.

A malicious user (Bob) could specify the shares to 99. As a result, the `assets` at Line 199 will be zero. Subsequently, the `_mintInternal` function at Line 201 will be executed with `assets=0, shares=99`.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L201

```solidity
File: wfCashERC4626.sol
198:     function mint(uint256 shares, address receiver) external override returns (uint256) {
199:         (uint256 assets, uint256 maxFCash) = _previewMint(shares); 
200:         // Will revert if matured 
201:         _mintInternal(assets, _safeUint88(shares), receiver, 0, maxFCash);
202:         emit Deposit(msg.sender, receiver, assets, shares);
203:         return assets;
204:     }
```

The `_mintInternal` will pull zero USDC from Bob's address and proceed to execute the `_lendLegacy` function at Line 69. Tracing the function call, the `NotionalV2.batchBalanceAndTradeAction => BatchActions.batchBalanceAndTradeAction => BatchActions._batchBalanceAndTradeAction` will be executed.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L69

```solidity
File: wfCashLogic.sol
35:     function _mintInternal(

..SNIP..
55:         } else {
56:             token.safeTransferFrom(msg.sender, address(this), depositAmountExternal); 
57:             depositAmountExternal = token.balanceOf(address(this)) - balanceBefore; 
58:         }

..SNIP..
68:         } else if (isETH || hasTransferFee || getCashBalance() > 0) { 
69:             _lendLegacy(currencyId, depositAmountExternal, fCashAmount, minImpliedRate, msgValue, isETH);
70:         } else {
71:             // Executes a lending action on Notional. Since this lending action uses an existing cash balance
72:             // prior to pulling payment, we cannot use it if there is a cash balance on the wrapper contract,
73:             // it will cause existing cash balances to be minted into fCash and create a shortfall. In normal
74:             // conditions, this method is more gas efficient.
75:             BatchLend[] memory action = EncodeDecode.encodeLendTrade(
76:                 currencyId,
77:                 getMarketIndex(),
78:                 fCashAmount,
79:                 minImpliedRate
80:             );
81:             NotionalV2.batchLend(address(this), action);
82:         }
```

Within the `BatchActions._batchBalanceAndTradeAction` function, it will first deposit zero USDC to the wrapper account, and zero cash (also called pCash/Prime Cash) is credited to the wrapper account when the `_executeDepositAction` function is executed. Only after the deposit action is executed, the trade will be executed via the `_executeTrades` function.

Note that the wrapper account can have a cash balance. Assume that the wrapper account has an existing positive cash balance. In this case, the existing cash balances will be used to mint into fCash. The fCash will be sent back to the wrapper contract, and wfCash will be minted to Bob.

Thus, Bob deposited zero assets but received wfCash in return. Notional is deployed on Arbitrum where the gas is cheap. As such, it is cost-effective to execute this attack multiple times.

## Impact

Cash within the wrapper account will be lost.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L114

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L201

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L69

## Tool used

Manual Review

## Recommendation

Consider implementing multiple controls to guard against such an attack:

1) If `assets` are zero, revert the transaction
2) With the first control alone, it might not be sufficient as rounding down assets will still happen (e.g. 1.9999 => 1), and malicious users could potentially take advantage of it. One defense is to implement an invariant check at the end of the minting process to ensure that the cash balance of the wrapper account never decreases after the minting is completed. For instance, `require(beforeCashBalance <= afterCashBalance)`. If this invariant is broken, this means that someone is exploiting the system, as minting should never cause one cash balance to decrease under any circumstance.