Alert Chartreuse Snail

medium

# Wfcash deposit might give incorrect amount of shares (wfcash) in some cases.

## Summary
Wfcash deposit might not give correct amount of wfcash when there is no fcash available on the markets.
## Vulnerability Detail
The preview function, which calculates the amount of share  for minting, does not take the edge case, when there is not available fcash on the markets, into account. In this case, amount of shares should be equivalent to the amount of asset (ratio 1:1). However, the code will get amount of share on notionalv2 instead, which is incorrect.
## Impact
Users will get less/ incorrect amount of shares when using this deposit function.
## Code Snippet
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L89-L106
## Tool used

Manual Review

## Recommendation
Consider adding code to handle the case when there is not enough available fcash on the markets.