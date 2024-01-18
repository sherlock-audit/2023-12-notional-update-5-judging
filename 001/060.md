Puny Gunmetal Giraffe

high

# The `wfCashLogic.mintViaUnderlying` function does not calculate residual token balance correctly

## Summary
The `wfCashLogic.mintViaUnderlying` function does not calculate residual token balance correctly, which leads to loss of funds.

## Vulnerability Detail
The `mintViaUnderlying` function calls `_mintInternal` which checks the balance before and after in `WETH` to determine how many residual tokens needs to be sent back to `msg.sender`:

[wfCashLogic.sol#L35-L92](https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L35-L92)
```solidity
    function _mintInternal(
        uint256 depositAmountExternal,
        uint88 fCashAmount,
        address receiver,
        uint32 minImpliedRate,
        uint256 maxFCash
    ) internal nonReentrant {
        require(!hasMatured(), "fCash matured");
        (IERC20 token, bool isETH, bool hasTransferFee, uint256 precision) = _getTokenForMintInternal();
        uint256 balanceBefore = isETH ? WETH.balanceOf(address(this)) : token.balanceOf(address(this));
        uint256 msgValue;
        uint16 currencyId = getCurrencyId();
        
        if (isETH) {
            // Use WETH if lending ETH. Although Notional natively supports ETH, we use WETH here for integration
            // contracts so they only have to support ERC20 token transfers.
            // NOTE: safeTransferFrom not required since WETH is known to be compatible
            IERC20((address(WETH))).transferFrom(msg.sender, address(this), depositAmountExternal);
            WETH.withdraw(depositAmountExternal);
            msgValue = depositAmountExternal;
        } else {
            token.safeTransferFrom(msg.sender, address(this), depositAmountExternal);
            depositAmountExternal = token.balanceOf(address(this)) - balanceBefore;
        }


        if (maxFCash < fCashAmount) {
            // NOTE: lending at zero
            uint256 fCashAmountExternal = fCashAmount * precision / uint256(Constants.INTERNAL_TOKEN_PRECISION);
            require(fCashAmountExternal <= depositAmountExternal);


            // NOTE: Residual (depositAmountExternal - fCashAmountExternal) will be transferred
            // back to the account
            NotionalV2.depositUnderlyingToken{value: msgValue}(address(this), currencyId, fCashAmountExternal);
        } else if (isETH || hasTransferFee || getCashBalance() > 0) {
            _lendLegacy(currencyId, depositAmountExternal, fCashAmount, minImpliedRate, msgValue, isETH);
        } else {
            // Executes a lending action on Notional. Since this lending action uses an existing cash balance
            // prior to pulling payment, we cannot use it if there is a cash balance on the wrapper contract,
            // it will cause existing cash balances to be minted into fCash and create a shortfall. In normal
            // conditions, this method is more gas efficient.
            BatchLend[] memory action = EncodeDecode.encodeLendTrade(
                currencyId,
                getMarketIndex(),
                fCashAmount,
                minImpliedRate
            );
            NotionalV2.batchLend(address(this), action);
        }


        // Mints ERC20 tokens for the receiver
        _mint(receiver, fCashAmount);


        // Residual tokens will be sent back to msg.sender, not the receiver. The msg.sender
        // was used to transfer tokens in and these are any residual tokens left that were not
        // lent out. Sending these tokens back to the receiver risks them getting locked on a
        // contract that does not have the capability to transfer them off
        _sendTokensToReceiver(token, msg.sender, isETH, balanceBefore);
    }
```

The "after" balance is determined in `_sendTokensToReceiver`:

[wfCashLogic.sol#L331-L346](https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L331-L346)
```solidity
    function _sendTokensToReceiver(
        IERC20 token,
        address receiver,
        bool isETH,
        uint256 balanceBefore
    ) private returns (uint256 tokensTransferred) {
        uint256 balanceAfter = isETH ? WETH.balanceOf(address(this)) : token.balanceOf(address(this));
        tokensTransferred = balanceAfter - balanceBefore;


        if (isETH) {
            // No need to use safeTransfer for WETH since it is known to be compatible
            IERC20(address(WETH)).transfer(receiver, tokensTransferred);
        } else if (tokensTransferred > 0) {
            token.safeTransfer(receiver, tokensTransferred);
        }
    }
```

(The following assumes `maxFCash < fCashAmount` in `_mintInternal`)
The problem with this function is that the difference of balance being used to determine the residual amount is in `WETH`, whereas the amount sent in `WETH` by the user is converted to `ETH` here:

[wfCashLogic.sol#L48-L54](https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L48-L54)
```solidity
        if (isETH) {
            // Use WETH if lending ETH. Although Notional natively supports ETH, we use WETH here for integration
            // contracts so they only have to support ERC20 token transfers.
            // NOTE: safeTransferFrom not required since WETH is known to be compatible
            IERC20((address(WETH))).transferFrom(msg.sender, address(this), depositAmountExternal);
            WETH.withdraw(depositAmountExternal);
            msgValue = depositAmountExternal;
```

Therefore `tokensTransferred` will always be 0, if the underlying sent is `ETH` and is never converted back to `WETH`.

[wfCashLogic.sol#L338](https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L338)
```solidity
        tokensTransferred = balanceAfter - balanceBefore;
```

## Impact
Users will lose residual tokens in the contract.

## Code Snippet
See above.

## Tool used

Manual Review

## Recommendation
I recommend calculating the balance before and after in terms of `ETH` instead of `WETH` and transfer the amount back to the user using a low-level call instead of tranferring `WETH`.