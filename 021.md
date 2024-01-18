Alert Chartreuse Snail

high

# wfCashLogic::_sellfCash doesn't work correctly

## Summary
wfCashLogic::_sellfCash does not calculate the amount of prime cash to be withdrawn correctly when there is insufficient fcash to sell 
## Vulnerability Detail
The amount of withdrawn prime cash  is calculated by the difference between cash balance before and after fcash trade. It is incorrect; The selling fcash (borrow trade) always transfer underlying token to wfcash contract, not cash/asset token, so the difference of balance before and after the trade is always 0. Thus, it never withdraw prime cash from notional.

Additionally, the wfcash contract withdraws prime cash because it does not have enough fcash itself. Therefore, amount of withdrawn prime cash should be calculated based on the difference between current fcash balance and the amount of fcash to be sold. Mathematically, it should be: primeCashToWithdraw = f(fCashToSell - fCashBalance)
There should be nothing to do with cash balance.

```solidity
   function _sellfCash(
        address receiver,
        uint256 fCashToSell,
        uint32 maxImpliedRate
    ) private returns (uint256 tokensTransferred) {
        (IERC20 token, bool isETH) = getToken(true);
        uint256 balanceBefore = isETH ? WETH.balanceOf(address(this)) : token.balanceOf(address(this));
        uint16 currencyId = getCurrencyId();

        (uint256 initialCashBalance, uint256 fCashBalance) = getBalances();
        bool hasInsufficientfCash = fCashBalance < fCashToSell;

        uint256 primeCashToWithdraw;
        if (hasInsufficientfCash) {
            // If there is insufficient fCash, calculate how much prime cash would be purchased if the
            // given fCash amount would be sold and that will be how much the wrapper will withdraw and
            // send to the receiver. Since fCash always sells at a discount to underlying prior to maturity,
            // the wrapper is guaranteed to have sufficient cash to send to the account.
            
            (/* */, primeCashToWithdraw, /* */, /* */) = NotionalV2.getPrincipalFromfCashBorrow(
                currencyId,
   >>>             fCashToSell,  //@audit incorrect amount, should be (fCashToSell - fCashBalance)
                getMaturity(),
                0,
                block.timestamp
            );
            // If this is zero then it signifies that the trade will fail.
            require(primeCashToWithdraw > 0, "Redeem Failed");

            // Re-write the fCash to sell to the entire fCash balance.
            fCashToSell = fCashBalance;

        }

        if (fCashToSell > 0) {
            // Sells fCash on Notional AMM (via borrowing)
            BalanceActionWithTrades[] memory action = EncodeDecode.encodeBorrowTrade(
                currencyId,
                getMarketIndex(),
                _safeUint88(fCashToSell),
                maxImpliedRate
            );
            NotionalV2.batchBalanceAndTradeAction(address(this), action);
        }

        uint256 postTradeCash = getCashBalance();

        // If the account did not have insufficient fCash, then the amount of cash change here is what
        // the receiver is owed. In the other case, we transfer to the receiver the total calculated amount
        // above without modification.
 >>>    if (!hasInsufficientfCash) primeCashToWithdraw = postTradeCash - initialCashBalance; //@audit - this should always be 0. Should be omitted
        require(primeCashToWithdraw <= postTradeCash);

        // Withdraw the total amount of cash and send it to the receiver
        NotionalV2.withdraw(currencyId, _safeUint88(primeCashToWithdraw), !isETH);
        tokensTransferred = _sendTokensToReceiver(token, receiver, isETH, balanceBefore);
    }
```
```solidity
    function encodeBorrowTrade(
        uint16 currencyId,
        uint8 marketIndex,
        uint88 fCashAmount,
        uint32 maxImpliedRate
    ) internal pure returns (BalanceActionWithTrades[] memory action) {
        action = new BalanceActionWithTrades[](1);
        action[0].actionType = DepositActionType.None;
        action[0].currencyId = currencyId;
        action[0].withdrawEntireCashBalance = false;
  >>>      action[0].redeemToUnderlying = true; //@audit - will always transfer underlying token to wfcash contract
        action[0].trades = new bytes32[](1);
        action[0].trades[0] = bytes32(
            (uint256(uint8(TradeActionType.Borrow)) << 248) |
            (uint256(marketIndex) << 240) |
            (uint256(fCashAmount) << 152) |
            (uint256(maxImpliedRate) << 120)
        );
    }
```
## Impact
Loss of funds for users.
## Code Snippet
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashLogic.sol#L275-L329

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/lib/EncodeDecode.sol#L82-L100

## Tool used

Manual Review

## Recommendation
Consider this change
```solidity
   function _sellfCash(
        address receiver,
        uint256 fCashToSell,
        uint32 maxImpliedRate
    ) private returns (uint256 tokensTransferred) {
        (IERC20 token, bool isETH) = getToken(true);
        uint256 balanceBefore = isETH ? WETH.balanceOf(address(this)) : token.balanceOf(address(this));
        uint16 currencyId = getCurrencyId();

        (uint256 initialCashBalance, uint256 fCashBalance) = getBalances();
        bool hasInsufficientfCash = fCashBalance < fCashToSell;

        uint256 primeCashToWithdraw;
        if (hasInsufficientfCash) {
            // If there is insufficient fCash, calculate how much prime cash would be purchased if the
            // given fCash amount would be sold and that will be how much the wrapper will withdraw and
            // send to the receiver. Since fCash always sells at a discount to underlying prior to maturity,
            // the wrapper is guaranteed to have sufficient cash to send to the account.
            
            (/* */, primeCashToWithdraw, /* */, /* */) = NotionalV2.getPrincipalFromfCashBorrow(
                currencyId,
                fCashToSell - fCashBalance, 
                getMaturity(),
                0,
                block.timestamp
            );
            // If this is zero then it signifies that the trade will fail.
            require(primeCashToWithdraw > 0, "Redeem Failed");

            // Re-write the fCash to sell to the entire fCash balance.
            fCashToSell = fCashBalance;

        }

        if (fCashToSell > 0) {
            // Sells fCash on Notional AMM (via borrowing)
            BalanceActionWithTrades[] memory action = EncodeDecode.encodeBorrowTrade(
                currencyId,
                getMarketIndex(),
                _safeUint88(fCashToSell),
                maxImpliedRate
            );
            NotionalV2.batchBalanceAndTradeAction(address(this), action);
        }

        uint256 postTradeCash = getCashBalance();

        // If the account did not have insufficient fCash, then the amount of cash change here is what
        // the receiver is owed. In the other case, we transfer to the receiver the total calculated amount
        // above without modification.

        // Withdraw the total amount of cash and send it to the receiver
        NotionalV2.withdraw(currencyId, _safeUint88(primeCashToWithdraw), !isETH);
        tokensTransferred = _sendTokensToReceiver(token, receiver, isETH, balanceBefore);
    }
```