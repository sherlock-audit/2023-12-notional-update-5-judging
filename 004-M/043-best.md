Strong Leather Toad

medium

# Unexpected behavior when calling certain ERC4626 functions

## Summary

Unexpected behavior could occur when certain ERC4626 functions are called during the time windows when the fCash has matured but is not yet settled.

## Vulnerability Detail

When the fCash has matured, the global settlement does not automatically get executed. The global settlement will only be executed when the first account attempts to settle its own account. The code expects the `pr.supplyFactor` to return zero if the global settlement has not been executed yet after maturity.

Per the comment at Line 215, the design of the `_getMaturedCashValue` function is that it expects that if fCash has matured AND the fCash has not yet been settled, the `pr.supplyFactor` will be zero. In this case, the cash value will be zero.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashBase.sol#L215

```solidity
File: wfCashBase.sol
209:     function _getMaturedCashValue(uint256 fCashAmount) internal view returns (uint256) { 
210:         if (!hasMatured()) return 0; 
211:         // If the fCash has matured we use the cash balance instead.
212:         (uint16 currencyId, uint40 maturity) = getDecodedID(); 
213:         PrimeRate memory pr = NotionalV2.getSettlementRate(currencyId, maturity); 
214: 
215:         // fCash has not yet been settled
216:         if (pr.supplyFactor == 0) return 0; 
..SNIP..
```

During the time window where the fCash has matured, and none of the accounts triggered an account settlement, the `_getMaturedCashValue` function at Line 33 below will return zero, which will result in the `totalAssets()` function returning zero.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L33

```solidity
File: wfCashERC4626.sol
29:     function totalAssets() public view override returns (uint256) {
30:         if (hasMatured()) {
31:             // We calculate the matured cash value of the total supply of fCash. This is
32:             // not always equal to the cash balance held by the wrapper contract.
33:             uint256 primeCashValue = _getMaturedCashValue(totalSupply());
34:             require(primeCashValue < uint256(type(int256).max));
35:             int256 externalValue = NotionalV2.convertCashBalanceToExternal(
36:                 getCurrencyId(), int256(primeCashValue), true
37:             );
38:             return externalValue >= 0 ? uint256(externalValue) : 0;
..SNIP..
```

## Impact

The `totalAssets()` function is utilized by key ERC4626 functions within the wrapper, such as the following functions. The side effects of this issue are documented below:

- `convertToAssets` (Impact = returned value is always zero assets regardless of the inputs)
- `convertToAssets` > `previewRedeem` (Impact = returned value is always zero assets regardless of the inputs)
- `convertToAssets` > `previewRedeem` > `maxWithdraw` (Impact = max withdrawal is always zero)
- `convertToShares` (Impact = Division by zero error, Revert)
- `convertToShares` > `previewWithdraw` (Impact = Revert)

In addition, any external protocol integrating with wfCash will be vulnerable within this time window as an invalid result (zero) is returned, or a revert might occur. For instance, any external protocol that relies on any of the above-affected functions for computing the withdrawal/minting amount or collateral value will be greatly impacted as the value before the maturity might be 10000, but it will temporarily reset to zero during this time window. Attackers could take advantage of this time window to perform malicious actions.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashBase.sol#L215

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L33

## Tool used

Manual Review

## Recommendation

Document the unexpected behavior of the affected functions that could occur during the time windows when the fCash has matured but is not yet settled so that anyone who calls these functions is aware of them.