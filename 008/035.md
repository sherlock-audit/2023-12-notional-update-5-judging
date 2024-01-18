Strong Leather Toad

high

# Overlending of assets to external market

## Summary

The current implementation does not take into consideration the interest earned by the protocol, causing the maximum amount of underlying assets that can be lent on the external market to be overestimated. As a result, overlending to external markets will occur, leading to liquidity issues where the protocol/users are unable to redeem its funds or carry out liquidation activities.

## Vulnerability Detail

 The `currentExternalUnderlyingLend` stores the amount of underlying assets lend to the external market (AAVE). Note that the `currentExternalUnderlyingLend` does not include the interest earned from the aToken's rebase.

The following is the extract from the [Audit Specification](https://docs.google.com/document/d/1-2iaTM8lBaurrfItOJRRveHnwKq1lEWGnewrEfXMzrI/edit) provided by the protocol team on the [contest page](https://audits.sherlock.xyz/contests/142) that describes the external withddraw threshold:

> External Withdraw Threshold: ensures that Notional has sufficient liquidity to withdraw from an external lending market. If Notional has 1000 units of underlying lent out on Aave, it requires 1000 * externalWithdrawThreshold units of underlying to be available on Aave for withdraw. This ensures there is sufficient buffer to process the redemption of Notional funds. If available liquidity on Aave begins to drop due to increased utilization, Notional will automatically begin to withdraw its funds from Aave to ensure that they are available for withdrawal on Notional itself.

Following is the implementation to compute the maximum amount of underlying assets that can be lent on the external market. Per the comment at 67 below, it mentioned that the max amount is a function of the excess redeemable funds on that market (funds that are redeemable in excess of Notional’s own funds on that market) and the `externalWithdrawThreshold`.

Focusing on the point "funds that are redeemable in excess of Notional’s own funds on that market". The issue is that the `currentExternalUnderlyingLend` does not represent the entire amount of Notional's own funds on the external market because it does not take into consideration the interest (aToken) held by Notional.

As such, it will underestimate the funds (aToken) held by Notional, which in turn overestimates the `maxExternalUnderlyingLend`, resulting in more external lending than expected.

For simplicity's sake, consider the simplified example:

- externalWithdrawThreshold = 100 USDC
- interest earned held by Notional = 100 USDC
- AAVE pool available liquidity for withdrawal = 200 USDC

In this case, the protocol should not proceed to lend out to the AAVE pool because there is no more redeemable liquidity left. However, since the protocol only used `externalWithdrawThreshold ` during the computation, it will mistaken that there is still a buffer of 100 USDC that is not owned by Notional, and proceed to lend externally until the threshold is hit. In the end, `Notional's own fund (externalWithdrawThreshold + interest earned held by Notional) > AAVE pool available liquidity for withdrawal`, causing issues when Notional redeems its funds.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/ExternalLending.sol#L61

```solidity
File: ExternalLending.sol
61:         // To ensure redeemability of Notional’s funds on external lending markets,
62:         // Notional requires there to be redeemable funds on the external lending market
63:         // that are a multiple of the funds that Notional has lent on that market itself.
64:         //
65:         // The max amount that Notional can lend on that market is a function
66:         // of the excess redeemable funds on that market
67:         // (funds that are redeemable in excess of Notional’s own funds on that market)
68:         // and the externalWithdrawThreshold.
69:         //
70:         // excessFunds = externalUnderlyingAvailableForWithdraw - currentExternalUnderlyingLend
71:         //
72:         // maxExternalUnderlyingLend * (externalWithdrawThreshold + 1) = maxExternalUnderlyingLend + excessFunds
73:         //
74:         // maxExternalUnderlyingLend * (externalWithdrawThreshold + 1) - maxExternalUnderlyingLend = excessFunds
75:         //
76:         // maxExternalUnderlyingLend * externalWithdrawThreshold = excessFunds
77:         //
78:         // maxExternalUnderlyingLend = excessFunds / externalWithdrawThreshold
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

## Impact

Notional will overlend its assets to the external market even if there is an inadequate amount of liquidity left in the AAVE pool available for withdrawal, leading to liquidity issues where the protocol/users are unable to redeem its funds or liquidation cannot be carried out, resulting in bad debt accumulating within the protocol and negatively affecting the protocol's solvency.

Also, it would be a serious issue if there is an emergency where funds need to be redeemed from AAVE quickly, but there is no or insufficient liquidity left in the AAVE pool to do so.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/ExternalLending.sol#L61

## Tool used

Manual Review

## Recommendation

When the Notional's own fund is used when computing the `maxExternalUnderlyingLend`, it should consist of the following assets for completeness:

- `currentExternalUnderlyingLend`
- Unskimmed Interest Earned
- Interest earned stored in Treasury contract (already skimmed earlier)