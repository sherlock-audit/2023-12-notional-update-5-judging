Short Candy Salmon

medium

# emissionRatePerYear invariant not match

## Summary
in `_accumulateRewardPerNToken()`
The invariant constraint of `emissionRatePerYear` is broken when `totalSupply == 0`.

## Vulnerability Detail
The document describes the invariant of `emissionRatePerYear`.
> The rewarder must emit a total of `emissionRatePerYear` tokens pro rata over any given time period, regardless of the number of nToken balance changes or ***the nToken total supply***.

However, in the method `_accumulateRewardPerNToken() -> _getAccumulatedRewardPerToken()`

```solidity
    function _accumulateRewardPerNToken(uint32 blockTime, uint256 totalSupply) private {
        // Ensure that end time is set to some value
        require(0 < endTime);
        uint32 time = uint32(SafeInt256.min(blockTime, endTime));

@>      accumulatedRewardPerNToken = _getAccumulatedRewardPerToken(time, totalSupply);

        lastAccumulatedTime = uint32(block.timestamp);
    }
```

when `totalSupply == 0`, `accumulatedRewardPerNToken` does not increase, 
but `lastAccumulatedTime` is updated to the current time.

This leads to, for example:
If no one `mints nToken` in the first month,
when someone `mints`, `accumulatedRewardPerNToken` will be 0, but `lastAccumulatedTime` will be updated to `February 1`.
So, in reality, only 11 months of `rewards token` are emitted.

According to the description of the invariant, it should still need to `emit`, giving all the rewards to the first person who mints.
This will quickly incentivize the first user to `mint ntoken`.

Or if all nTokens are `burned` in the middle, this problem will also occur.

## Impact

The invariant of `emissionRatePerYear` does not match

## Code Snippet
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/adapters/SecondaryRewarder.sol#L260-L268
## Tool used

Manual Review

## Recommendation
```diff
    function _accumulateRewardPerNToken(uint32 blockTime, uint256 totalSupply) private {
        // Ensure that end time is set to some value
        require(0 < endTime);
+      if (totalSupply) return; // return for reward the first user, previously aggregated
        uint32 time = uint32(SafeInt256.min(blockTime, endTime));

        accumulatedRewardPerNToken = _getAccumulatedRewardPerToken(time, totalSupply);

        lastAccumulatedTime = uint32(block.timestamp);
    }
```
