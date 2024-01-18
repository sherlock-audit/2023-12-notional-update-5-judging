Strong Leather Toad

high

# Malicious users could block liquidation or perform DOS

## Summary

The current implementation uses a "push" approach where reward tokens are sent to the recipient during every update, which introduces additional attack surfaces that the attackers can exploit. An attacker could intentionally affect the outcome of the transfer to gain a certain advantage or carry out certain attack.

The worst-case scenario is that malicious users might exploit this trick to intentionally trigger a revert when someone attempts to liquidate their unhealthy accounts to block the liquidation, leaving the protocol with bad debts and potentially leading to insolvency if it accumulates.

## Vulnerability Detail

Per the [Audit Scope Documentation](https://docs.google.com/document/d/1-2iaTM8lBaurrfItOJRRveHnwKq1lEWGnewrEfXMzrI/edit) provided by the protocol team on the [contest page](https://audits.sherlock.xyz/contests/142), the reward tokens can be any arbitrary ERC20 tokens

> We are extending this functionality to allow nTokens to be incentivized by a secondary reward token. On Arbitrum, this will be ARB as a result of the ARB STIP grant. In the future, this may be any arbitrary ERC20 token

Line 231 of the `_claimRewards` function below might revert due to various issues such as:

- tokens with blacklisting features such as USDC (users might intentionally get into the blacklist to achieve certain outcomes)
- tokens with hook, which allow the target to revert the transaction intentionally
- unexpected error in the token's contract

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/adapters/SecondaryRewarder.sol#L231

```solidity
File: SecondaryRewarder.sol
216:     function _claimRewards(address account, uint256 nTokenBalanceBefore, uint256 nTokenBalanceAfter) private { 
217:         uint256 rewardToClaim = _calculateRewardToClaim(account, nTokenBalanceBefore, accumulatedRewardPerNToken); 
218: 
219:         // Precision here is:
220:         //  nTokenBalanceAfter (INTERNAL_TOKEN_PRECISION) 
221:         //  accumulatedRewardPerNToken (INCENTIVE_ACCUMULATION_PRECISION) 
222:         // DIVIDE BY
223:         //  INTERNAL_TOKEN_PRECISION 
224:         //  => INCENTIVE_ACCUMULATION_PRECISION (1e18) 
225:         rewardDebtPerAccount[account] = nTokenBalanceAfter 
226:             .mul(accumulatedRewardPerNToken)
227:             .div(uint256(Constants.INTERNAL_TOKEN_PRECISION))
228:             .toUint128(); 
229: 
230:         if (0 < rewardToClaim) { 
231:             GenericToken.safeTransferOut(REWARD_TOKEN, account, rewardToClaim); 
232:             emit RewardTransfer(REWARD_TOKEN, account, rewardToClaim);
233:         }
234:     }
```

If a revert occurs, the following functions are affected:

```solidity
_claimRewards -> claimRewardsDirect

_claimRewards -> claimRewards -> Incentives.claimIncentives
_claimRewards -> claimRewards -> Incentives.claimIncentives -> BalancerHandler._finalize
_claimRewards -> claimRewards -> Incentives.claimIncentives -> BalancerHandler._finalize -> Used by many functions

_claimRewards -> claimRewards -> Incentives.claimIncentives -> BalancerHandler.claimIncentivesManual
_claimRewards -> claimRewards -> Incentives.claimIncentives -> BalancerHandler.claimIncentivesManual -> nTokenAction.nTokenClaimIncentives (External)
_claimRewards -> claimRewards -> Incentives.claimIncentives -> BalancerHandler.claimIncentivesManual -> nTokenAction.nTokenClaimIncentives (External) -> claimNOTE (External)
```

## Impact

Many of the core functionalities of the protocol will be affected by the revert. Specifically, the `BalancerHandler._finalize` has the most impact as this function is called by almost every critical functionality of the protocol, including deposit, withdrawal, and liquidation. 

The worst-case scenario is that malicious users might exploit this trick to intentionally trigger a revert when someone attempts to liquidate their unhealthy accounts to block the liquidation, leaving the protocol with bad debts and potentially leading to insolvency if it accumulates.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/adapters/SecondaryRewarder.sol#L231

## Tool used

Manual Review

## Recommendation

The current implementation uses a "push" approach where reward tokens are sent to the recipient during every update, which introduces additional attack surfaces that the attackers can exploit.

Consider adopting a pull method for users to claim their rewards instead so that the transfer of reward tokens is disconnected from the updating of reward balances.