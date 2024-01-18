Tame Beige Pigeon

medium

# Hardcoded Value of minImpliedRate in wfCashERC4626

## Summary
In wfCashERC4626.sol, both the mint() and deposit() functions contains a hardcoded value for minImpliedRate. This implementation may result in users unknowingly accepting a yield of 0%.

## Vulnerability Detail
The vulnerability is present in the mint(uint256 shares, address receiver) function. Within its execution, another call to _mintInternal(..., 0, ...) is made, where the value of minImpliedRate is hardcoded as 0. This hardcoded value signifies that users are implicitly willing to accept a 0% interest rate. In the event of any issues with the project, this can have serious consequences for users.
The identical issue mentioned above is also present in the deposit() function.

## Impact
Users unintentionally accept a yield of 0% due to the hardcoded minImpliedRate. 


## Code Snippet
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L189-L195

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L198-L204

## Tool used

Manual Review

## Recommendation
I  recommended to refactor the mint() and deposit() functions in wfCashERC4626.sol to allow for dynamic specification of minImpliedRate. Instead of hardcoding the value, consider making it a parameter of the function or sourcing it from a configurable setting. 