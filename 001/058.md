Sweet Blood Cod

high

# `wfCashERC4626` - Lending at 0% interest with a fee-on-transfer asset makes vault insolvent

## Summary
The `wfCash` vault is credited less prime cash than the `wfCash` it mints to the depositor when its underlying asset is a fee-on-transfer token. This leads to the vault being insolvent because it has issued more shares than can be redeemed. 

## Vulnerability Detail
When minting `wfCash` shares more than the `maxFCash` available, the `wfCashERC4626` vault lends the deposited assets at 0% interest. The assets are deposited by the vault into Notional and the depositor gets 1:1 `wfCash` in return. This works fine for assets that are not fee-on-transfer tokens. 
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L60-L68
```solidity
    if (maxFCash < fCashAmount) {
        // NOTE: lending at zero
        uint256 fCashAmountExternal = fCashAmount * precision / uint256(Constants.INTERNAL_TOKEN_PRECISION);
        require(fCashAmountExternal <= depositAmountExternal);

        // NOTE: Residual (depositAmountExternal - fCashAmountExternal) will be transferred
        // back to the account
        NotionalV2.depositUnderlyingToken{value: msgValue}(address(this), currencyId, fCashAmountExternal);
    } else if (isETH || hasTransferFee || getCashBalance() > 0) {
```
For fee-on-transfer tokens, the vault is credited prime cash based on the actual amount it received, which is `deposit amount - transfer fee`. 
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/TokenHandler.sol#L204-L214
```solidity
        } else {
            // In the case of deposits, we use a balance before and after check
            // to ensure that we record the proper balance change.
            actualTransferExternal = GenericToken.safeTransferIn(
                underlying.tokenAddress, account, underlyingExternalDeposit
            ).toInt();
        }

        netPrimeSupplyChange = _postTransferPrimeCashUpdate(
            account, currencyId, actualTransferExternal, underlying, primeRate
        );
```

However, the vault mints `fCashAmount` of shares to the depositor. 
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L84-L85
```solidity
    // Mints ERC20 tokens for the receiver
    _mint(receiver, fCashAmount);
```

In the case of lending at 0% interest, `fCashAmount` is equal to `depositAmount` but at 1e8 precision. 

To simplify the example, let us assume that there are no other depositors. When the sole depositor redeems all their `wfCash` shares at maturity, they will be unable to redeem all their shares because the `wfCash` vault does not hold enough prime cash.

## Impact
Although the example used to display the vulnerability is for the case of lending at 0% interest, the issue exists for minting any amount of shares. 

The `wfCashERC4626` vault will become insolvent and unable to buy back all shares. The larger the total amount deposited, the larger the deficit. The deficit is equal to the transfer fee. Given a total deposit amount of 100M USDT and a transfer fee of 2% (assuming a transfer fee was set and enabled for USDT), 2M USDT will be the deficit. 

The last depositors to redeem their shares will be shouldering the loss.

## Code Snippet
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L60-L68
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/internal/balances/TokenHandler.sol#L204-L214
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L84-L85

## Tool used
Manual Review

## Recommendation
Consider adding the following:
1. A flag in `wfCashERC4626` that signals that the vault's asset is a fee-on-transfer token. 
2. In `wfCashERC4626._previewMint()` and `wfCashERC46262._previewDeposit`, all calculations related to `assets` should account for the transfer fee of the token.