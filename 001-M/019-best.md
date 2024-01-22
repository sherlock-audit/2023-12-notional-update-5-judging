Short Candy Salmon

medium

# recover() using the standard transfer may not be able to retrieve some tokens

## Summary
in `SecondaryRewarder.recover()`
Using the standard `IERC20.transfer()`
If `REWARD_TOKEN` is like `USDT`, it will not be able to transfer out, because this kind of `token` does not return `bool`
This will cause it to always `revert`

## Vulnerability Detail
`SecondaryRewarder.recover()` use for 

> Allows the Notional owner to recover any tokens sent to the address or any reward tokens remaining on the contract in excess of the total rewards emitted.

```solidity
    function recover(address token, uint256 amount) external onlyOwner {
        if (Constants.ETH_ADDRESS == token) {
            (bool status,) = msg.sender.call{value: amount}("");
            require(status);
        } else {
@>          IERC20(token).transfer(msg.sender, amount);
        }
    }
```
Using the standard `IERC20.transfer()` method to execute the transfer
A `token` of a type similar to `USDT` has no return value
This will cause the execution of the transfer to always fail

## Impact

If `REWARD_TOKEN` is like `USDT`, it will not be able to transfer out.

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/adapters/SecondaryRewarder.sol#L152C3-L159

## Tool used

Manual Review

## Recommendation
```diff
    function recover(address token, uint256 amount) external onlyOwner {
        if (Constants.ETH_ADDRESS == token) {
            (bool status,) = msg.sender.call{value: amount}("");
            require(status);
        } else {
-          IERC20(token).transfer(msg.sender, amount);
+          GenericToken.safeTransferOut(token,msg.sender,amount);
        }
    }
```