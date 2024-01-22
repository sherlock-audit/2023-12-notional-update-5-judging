Strong Leather Toad

medium

# Does not approve to zero for USDT

## Summary

USDT requires resetting the approval to 0 first before being able to reset it to another value. Otherwise, it will revert and Notional will not be able to lend USDT to the external market.

## Vulnerability Detail

Per the contest page (https://audits.sherlock.xyz/contests/142), it mentioned the following:

> Do you expect to use any of the following tokens with non-standard behaviour with the smart contracts?
>
> USDC and USDT are the primary examples.

Thus, it is expected that the AAVE oracle will interact with USDT.

If there is residual approval from a previous transaction, USDT requires resetting the approval to 0 first before being able to reset it to another value. If the `UNDERLYING_TOKEN` is USDT, it must be approved to zero first. 

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/pCash/AaveV3HoldingsOracle.sol#L123

```solidity
File: AaveV3HoldingsOracle.sol
121:         } else {
122:             targets[0] = UNDERLYING_TOKEN;
123:             callData[0] = abi.encodeWithSelector(IERC20.approve.selector, LENDING_POOL, depositUnderlyingAmount);
124: 
125:             targets[1] = LENDING_POOL;
126:             callData[1] = abi.encodeWithSelector(
127:                 ILendingPool.deposit.selector,
128:                 UNDERLYING_TOKEN,
129:                 depositUnderlyingAmount,
130:                 from,
131:                 0 // referralCode
132:             );
133:         }
```

## Impact

Notional will not be able to lend USDT to the external market. One of the key benefits of prime cash is its ability to lend unused underlying tokens to the external market to earn increased yield. If the external lending feature is not working as intended, Notional and its users will lose the yields that can be earned from the external markets.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/pCash/AaveV3HoldingsOracle.sol#L123

## Tool used

Manual Review

## Recommendation

Consider approving to 0 first before setting it to another value. 