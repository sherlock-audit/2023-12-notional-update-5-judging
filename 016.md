Alert Chartreuse Snail

high

# Malicious users might abuse lending at zero mechanism to get more shares at the cost of other users minting after them

## Summary
Malicious users might abuse lending at zero mechanism to get more shares at the cost of other users minting after them.
## Vulnerability Detail
Lending at zero allows users to mint shares for 1:1 ratio when there is not enough available fcash on markets (mint fcash amount > avail fcash). 

The problem is that lending at zero allows user to mint all their shares for prime cash while there is still some available fcash in markets. A malicious user will try to mint more shares (wfcash) than avail fcash in order to get good rate; user joining after will eat the loss if their amount of minted shares is less than available fcash. In other words, they will have to pay exchange fee and endure exchange rate for the previous user.

Consider this scenario:
- There is 300 available fcash on the markets.
- User A mints 301 shares. He get good 1:1 ratio because 301 > 300 so the vault will do lending at zero.
- User B comes in and mints 150 shares. Since 150 < 300, he has to pay more asset to cover exchange rate and fee.

It's clearly unfair for user B; he has to pay more while getting no extra profit.

Another impact is that it makes the system inconsistent, for same actions could lead to different results.
For example, when there is 300 available fcash on the markets, minting 600 shares  and minting 300 shares twice lead to different outcomes. In the first action, 600 fcash will go into prime cash borrowing. However, in the second action, 300fcash will go into prime cash borrowing, and 300fcash will go into fcash lending. 
## Impact
Malicious users can gain profit at the cost of other users.
## Code Snippet
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashERC4626.sol#L120-L134

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L60-L82
## Tool used

Manual Review

## Recommendation
Consider making a change to lending at zero mechanism. When the amount of minted shares is greater than available fcash, then all available fcash is going into fcash lending; the rest (minted shares - avail fcash) is going into prime borrowing.