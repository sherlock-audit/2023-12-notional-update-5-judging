Strong Leather Toad

high

# External lending can exceed the threshold

## Summary

Due to an incorrect calculation of the max lending amount, external lending can exceed the external withdrawal threshold. If this restriction/threshold is not adhered to, users or various core functionalities within the protocol will have issues redeeming or withdrawing their prime cash.

## Vulnerability Detail

The following is the extract from the [Audit Scope Documentation](https://docs.google.com/document/d/1-2iaTM8lBaurrfItOJRRveHnwKq1lEWGnewrEfXMzrI/edit) provided by the protocol team on the [contest page](https://audits.sherlock.xyz/contests/142) that describes the external withdraw threshold:

> ●	External Withdraw Threshold: ensures that Notional has sufficient liquidity to withdraw from an external lending market. If Notional has 1000 units of underlying lent out on Aave, it requires 1000 * externalWithdrawThreshold units of underlying to be available on Aave for withdraw. This ensures there is sufficient buffer to process the redemption of Notional funds. If available liquidity on Aave begins to drop due to increased utilization, Notional will automatically begin to withdraw its funds from Aave to ensure that they are available for withdrawal on Notional itself.

To ensure the redeemability of Notional’s funds on external lending markets, Notional requires there to be redeemable funds on the external lending market that are a multiple of the funds that Notional has lent on that market itself.

Assume that the `externalWithdrawThreshold` is 200% and the underlying is USDC. Therefore, `PERCENTAGE_DECIMALS/externalWithdrawThreshold = 100/200 = 0.5` (Line 83-84 below). This means that the number of USDC to be available on AAVE for withdrawal must be two (2) times the number of USDC Notional lent out on AAVE (A multiple of 2).

The `externalUnderlyingAvailableForWithdraw` stores the number of liquidity in USDC on the AAVE pool available to be withdrawn.

If `externalUnderlyingAvailableForWithdraw` is 1000 USDC and `currentExternalUnderlyingLend` is 400 USDC, this means that the remaining 600 USDC liquidity on the AAVE pool is not owned by Notional. 

The `maxExternalUnderlyingLend` will be  `600 * 0.5 = 300`. Thus, the maximum amount that Notional can lend externally at this point is 300 USDC.

Assume that after Notional has lent 300 USDC externally to the AAVE pool.

The `currentExternalUnderlyingLend` will become `400+300=700`, and the `externalUnderlyingAvailableForWithdraw` will become `1000+300=1300`

Following is the percentage of USDC in AAVE that belong to Notional

```solidity
700/1300 = 0.5384615385 (53%).
```

At this point, the invariant is broken as the number of USDC to be available on AAVE for withdrawal is less than two (2) times the number of USDC lent out on AAVE after the lending.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/ExternalLending.sol#L81

```solidity
File: ExternalLending.sol
36:     function getTargetExternalLendingAmount(
..SNIP..
79:         uint256 maxExternalUnderlyingLend;
80:         if (oracleData.currentExternalUnderlyingLend < oracleData.externalUnderlyingAvailableForWithdraw) {
81:             maxExternalUnderlyingLend =
82:                 (oracleData.externalUnderlyingAvailableForWithdraw - oracleData.currentExternalUnderlyingLend)
83:                 .mul(uint256(Constants.PERCENTAGE_DECIMALS))
84:                 .div(rebalancingTargetData.externalWithdrawThreshold);
85:         } else {
86:             maxExternalUnderlyingLend = 0;
87:         }
```

The root cause is that when USDC is deposited to AAVE to get aUSDC, the total USDC in the pool increases. Therefore, using the current amount of USDC in the pool to determine the maximum deposit amount is not an accurate measure of liquidity risk.

## Impact

To ensure the redeemability of Notional’s funds on external lending markets, Notional requires there to be redeemable funds on the external lending market that are a multiple of the funds that Notional has lent on that market itself.

If this restriction is not adhered to, users or various core functionalities within the protocol will have issues redeeming or withdrawing their prime cash. For instance, users might not be able to withdraw their assets from the protocol due to insufficient liquidity, or liquidation cannot be carried out due to lack of liquidity, resulting in bad debt accumulating within the protocol and negatively affecting the protocol's solvency.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/ExternalLending.sol#L81

## Tool used

Manual Review

## Recommendation

To ensure that a deposit does not exceed the threshold, the following formula should be used to determine the maximum deposit amount:

Let's denote:

$T$ as the externalWithdrawThreshold
$L$ as the currentExternalUnderlyingLend
$W$ as the externalUnderlyingAvailableForWithdraw
$D$ as the Deposit (the variable we want to solve for)

$$
T = \frac{L + D}{W + D}
$$

Solving $D$, the formula for calculating the maximum deposit ($D$) is

$$
D = \frac{TW-L}{1-T}
$$

Using back the same example in the "Vulnerability Detail" section.

The maximum deposit amount is as follows:

```solidity
D = (TW - L) / (1 - T)
D = (0.5 * 1000 - 400) / (1 - 0.5)
D = (500 - 400) / 0.5 = 200
```

If 200 USDC is lent out, it will still not exceed the threshold of 200%, which demonstrates that the formula is working as intended in keeping the multiple of two (200%) constant before and after the deposit.

```solidity
(400 + 200) / (1000 + 200) = 0.5
```