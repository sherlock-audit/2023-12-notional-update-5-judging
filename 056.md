Winning Eggplant Owl

medium

# Rounding issue on `previewMint` and `previewWithdraw`

## Summary

Rounding issue on `previewMint` and `previewWithdraw` which doesn't follow EIP 4626's Security Considerations.

## Vulnerability Detail

Per EIP 4626's Security Considerations (https://eips.ethereum.org/EIPS/eip-4626)

> Finally, EIP-4626 Vault implementers should be aware of the need for specific, opposing rounding directions across the different mutable and view methods, as it is considered most secure to favor the Vault itself during calculations over its users:

> If (1) it’s calculating how many shares to issue to a user for a certain amount of the underlying tokens they provide or (2) it’s determining the amount of the underlying tokens to transfer to them for returning a certain amount of shares, it should round down.

> If (1) it’s calculating the amount of shares a user has to supply to receive a given amount of the underlying tokens or (2) it’s calculating the amount of underlying tokens a user has to provide to receive a certain amount of shares, it should round up.

Thus, the result of the `previewMint` and `previewWithdraw` should be rounded up.

ERC4626 expects the result returned from `previewWithdraw` function to be rounded up. However, within the `previewWithdraw` function, it calls the `convertToShares` function. Meanwhile, the `convertToShares` function returned a rounded down value, thus `previewWithdraw` will return a rounded down value instead of round up value. Thus, this function does not behave as expected.

The same case goes to `previewMint`.

```js
File: wfCashERC4626.sol
46:     function convertToShares(uint256 assets) public view override returns (uint256 shares) {
47:         uint256 supply = totalSupply();
48:         if (supply == 0) {
49:             // Scales assets by the value of a single unit of fCash
50:             (/* */, uint256 unitfCashValue) = _getPresentCashValue(uint256(Constants.INTERNAL_TOKEN_PRECISION));
51:             return (assets * uint256(Constants.INTERNAL_TOKEN_PRECISION)) / unitfCashValue;
52:         }
53: 
54:         return (assets * supply) / totalAssets();
55:     }
...
143:     function previewWithdraw(uint256 assets) public view override returns (uint256 shares) {
144:         if (assets == 0) return 0;
...
153:         if (hasMatured()) {
154:             shares = convertToShares(assets);
155:         } else {
156:             // If withdrawing non-matured assets, we sell them on the market (i.e. borrow)
157:             (uint16 currencyId, uint40 maturity) = getDecodedID();
158:             (shares, /* */, /* */) = NotionalV2.getfCashBorrowFromPrincipal(
159:                 currencyId,
160:                 assets,
161:                 maturity,
162:                 0,
163:                 block.timestamp,
164:                 true
165:             );
166:         }
167:     }
```

## Impact

Other protocols that integrate with wfCash might wrongly assume that the functions handle rounding as per ERC4626 expectation. Thus, it might cause some intergration problem in the future that can lead to wide range of issues for both parties.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L46-L55

## Tool used

Manual Review

## Recommendation

Ensure that the rounding of vault's functions behave as expected. 