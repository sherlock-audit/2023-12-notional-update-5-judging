Strong Leather Toad

high

# Users can borrow beyond the debt cap

## Summary

The purpose of the debt cap is to ensure the redeemability of prime cash. It was found that users can borrow beyond the debt cap set by Notional, resulting in users or various core functionalities within the protocol having issues redeeming or withdrawing their prime cash if the debt cap is exceeded.

## Vulnerability Detail

At T1, assume that Bob's account `storedCashBalance` has 50 cash, and `totalUnderlyingDebt` is 700 USDC

At T2, for simplicity's sake, assume the following values at this point:

- the `maxUnderlyingDebt` is 1000 USDC
- the exchange rate of prime cash and underlying is 1:1
- the `totalUnderlyingDebt` progressively grows from 700 USDC to 990 USDC due to various trading or liquidation activities. Thus, 10 USDC more can be borrowed before hitting the debt cap of 1000 USDC

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/pCash/PrimeSupplyCap.sol#L35

```solidity
File: PrimeSupplyCap.sol
35:     function checkDebtCap(PrimeRate memory pr, uint16 currencyId) internal view { 
36:         (
37:             /* */, /* */,
38:             uint256 maxUnderlyingDebt,
39:             uint256 totalUnderlyingDebt
40:         ) = getSupplyCap(pr, currencyId);
41:         if (maxUnderlyingDebt == 0) return;
42: 
43:         require(totalUnderlyingDebt <= maxUnderlyingDebt, "Over Debt Cap");
44:     }
```

Note that Bob does not have any debt and also does not own any of the 990 USDC debt (`totalUnderlyingDebt`).

At T3, Bob decided to borrow 40 pCash (variable). The condition at Line 177 below is as follows during borrowing and will be evaluated to `false`:

```solidity
checkAllowPrimeBorrow && totalCashChange < 0 && balanceState.storedCashBalance.add(totalCashChange) < 0
true && -40 < 0 && 50.add(-40) < 0
true && true && false
false
```

The `checkDebtCap = true` code at Line 196 will not be executed, and `checkDebtCap` will remain uninitialized and `false`. As such, the debt cap check will not be performed. After Bob's borrowing, the `totalUnderlyingDebt` increased by 40 from 990 USDC to 1030 USDC, which exceeded the debt cap of 1000 USDC.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/BalanceHandler.sol#L196

```solidity
File: BalanceHandler.sol
151:     function  _finalize(
..SNIP..
157:         bool checkAllowPrimeBorrow
158:     ) private returns (int256 transferAmountExternal) {
..SNIP..
174:             // No changes to total cash after this point
175:             int256 totalCashChange = balanceState.netCashChange.add(balanceState.primeCashWithdraw);
176: 
177:             if (
178:                 checkAllowPrimeBorrow &&
179:                 totalCashChange < 0 &&
180:                 balanceState.storedCashBalance.add(totalCashChange) < 0
181:             ) {

..SNIP..
195:                 require(accountContext.allowPrimeBorrow, "No Prime Borrow");
196:                 checkDebtCap = true;
197:             }
..SNIP..
File: BalanceHandler.sol
245:         if (checkDebtCap) balanceState.primeRate.checkDebtCap(balanceState.currencyId);
246:     }
```

## Impact

The purpose of the debt cap is to ensure the redeemability of prime cash and ensure sufficient liquidity for withdraws and liquidations. If the debt cap is not properly enforced, users or various core functionalities within the protocol will encounter issues redeeming or withdrawing their prime cash. For instance, users might not be able to withdraw their assets from the protocol due to insufficient liquidity, or liquidation cannot be carried out due to lack of liquidity, resulting in bad debt accumulating within the protocol and negatively affecting the protocol's solvency.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/pCash/PrimeSupplyCap.sol#L35

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/BalanceHandler.sol#L196

## Tool used

Manual Review

## Recommendation

The debt cap check should be performed as long as the user borrows any cash regardless of whether the condition `balanceState.storedCashBalance.add(totalCashChange) < 0` is true or false.