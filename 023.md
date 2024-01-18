Strong Leather Toad

high

# Loss of rewards if an update is triggered too close to the last update

## Summary

If an update is triggered too close to the last update, it will lead to a loss of rewards.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/global/Constants.sol#L11

```solidity
File: Constants.sol
11:     uint256 internal constant INCENTIVE_ACCUMULATION_PRECISION = 1e18;
```

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/adapters/SecondaryRewarder.sol#L270

```solidity
function _calculateRewardToClaim(address account, uint256 nTokenBalanceAtLastClaim, uint128 rewardsPerNToken) 
	private
	view
	returns (uint256)
{
	// Precision here is:
	//   nTokenBalanceAtLastClaim (INTERNAL_TOKEN_PRECISION)
	//   mul rewardsPerNToken (INCENTIVE_ACCUMULATION_PRECISION)
	//   div INTERNAL_TOKEN_PRECISION
	// => INCENTIVE_ACCUMULATION_PRECISION
	// SUB rewardDebtPerAccount (INCENTIVE_ACCUMULATION_PRECISION)
	//
	// - mul REWARD_TOKEN_DECIMALS
	// - div INCENTIVE_ACCUMULATION_PRECISION
	// => REWARD_TOKEN_DECIMALS
	return uint256(nTokenBalanceAtLastClaim)
		.mul(rewardsPerNToken)
		.div(uint256(Constants.INTERNAL_TOKEN_PRECISION))
		.sub(rewardDebtPerAccount[account])
		.mul(10 ** REWARD_TOKEN_DECIMALS)
		.div(Constants.INCENTIVE_ACCUMULATION_PRECISION);
}
```

Following is the formula used within the `_calculateRewardToClaim` function above:

$$
\frac{\frac{( nTokenBalanceAtLastClaim \times rewardsPerNToken)}{Constants.INTERNAL\_TOKEN\_PRECISION} - rewardDebtPerAccount) \times 10^{REWARD\_TOKEN\_DECIMALS}}{Constants.INCENTIVE\_ACCUMULATION\_PRECISION}
$$

If the `_calculateRewardToClaim` function is triggered shortly after the last reward claim, the $\frac{( nTokenBalanceAtLastClaim \times rewardsPerNToken)}{Constants.INTERNAL\_TOKEN\_PRECISION}$ portion would not be much larger than the $rewardDebtPerAccount$. This is because if only a short time has passed, the accumulated reward per nToken ($rewardsPerNToken$) will only increase by a tiny amount.

When these two variables are subtracted, it will result in a small value called $s$.

Assume that the reward token is USDC (6 decimals precision). In this case, the amount of rewards to claim is as follows:

$$
rewardToClaim = \frac{( s \times 10^6)}{10^{18}}
$$

If $s < 10^{12}$, the $rewardToClaim$ will round down to zero due to the Solidity rounding error.

As a result of the rounding to zero error, the user who triggered it will lose the claim to the reward. In Line 217 below, the `rewardToClaim` will be set to zero, and no reward tokens will be transferred to the user. Although no reward tokens are sent, the `rewardDebtPerAccount[account]` of the victim still gets updated to the latest accumulated reward rate.

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/adapters/SecondaryRewarder.sol#L216

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

## Impact

Loss of reward tokens.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/adapters/SecondaryRewarder.sol#L270

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/adapters/SecondaryRewarder.sol#L216

## Tool used

Manual Review

## Recommendation

Consider updating the `rewardDebtPerAccount[account]` of the user only if reward tokens are sent. In this case, even if the `rewardToClaim` rounds down to zero, the user will not lose any reward tokens. The users could claim the reward again at a much later time.

```diff
function _claimRewards(address account, uint256 nTokenBalanceBefore, uint256 nTokenBalanceAfter) private { 
	uint256 rewardToClaim = _calculateRewardToClaim(account, nTokenBalanceBefore, accumulatedRewardPerNToken); 

-	// Precision here is:
-	//  nTokenBalanceAfter (INTERNAL_TOKEN_PRECISION) 
-	//  accumulatedRewardPerNToken (INCENTIVE_ACCUMULATION_PRECISION) 
-	// DIVIDE BY
-	//  INTERNAL_TOKEN_PRECISION 
-	//  => INCENTIVE_ACCUMULATION_PRECISION (1e18) 
-	rewardDebtPerAccount[account] = nTokenBalanceAfter 
-		.mul(accumulatedRewardPerNToken)
-		.div(uint256(Constants.INTERNAL_TOKEN_PRECISION))
-		.toUint128(); 

	if (0 < rewardToClaim) { 	
+	// Precision here is:
+	//  nTokenBalanceAfter (INTERNAL_TOKEN_PRECISION) 
+	//  accumulatedRewardPerNToken (INCENTIVE_ACCUMULATION_PRECISION) 
+	// DIVIDE BY
+	//  INTERNAL_TOKEN_PRECISION 
+	//  => INCENTIVE_ACCUMULATION_PRECISION (1e18) 
+	rewardDebtPerAccount[account] = nTokenBalanceAfter 
+		.mul(accumulatedRewardPerNToken)
+		.div(uint256(Constants.INTERNAL_TOKEN_PRECISION))
+		.toUint128(); 

		GenericToken.safeTransferOut(REWARD_TOKEN, account, rewardToClaim); 
		emit RewardTransfer(REWARD_TOKEN, account, rewardToClaim);
	}
}
```