Bouncy Fiery Shell

medium

# Precision Loss in  convertToAssets() Function due to Decimal Normalization

## Summary

The contract `BaseERC4626Proxy` has a precision loss issue in the `convertToAssets()` function. The contract performs `division before multiplication (Decimal Normalization)`, which results in a loss of precision. 

## Vulnerability Detail

Solidity uses integer division, which discards the remainder and can lead to a loss of precision. This means that if the result of the division operation is a decimal number, it will be rounded down to the nearest whole number, losing any fractional part.

The precision loss vulnerability in the `convertToAssets` function arises from the order of operations in the expression ` return shares.mul(exchangeRate).div(EXCHANGE_RATE_PRECISION).div(truncate).mul(truncate)`. 


In this function, `shares` is multiplied by `exchangeRate()`, then the result is divided by `EXCHANGE_RATE_PRECISION`. After that, the result is divided again by `truncate` and then multiplied by `truncate`.


### Scenario:

Let's consider `shares is 1000, exchangeRate is 2, EXCHANGE_RATE_PRECISION is 1e18, and truncate is 1000`.

The original calculation would be:

```Solidity
(1000 * 2) / 1e18 / 1000 * 1000 = 2 
```


Now, let's rearrange the operations to perform all multiplications before divisions:

```Solidity
(1000 * 2 * 1e18 * 1000) / 1e18 / 1000 = 4e18 
```

Comparing the two outputs, we can see that the reordered equation gives a much larger result, demonstrating the importance of the order of operations in preserving precision


## Impact

The final value returned by the `convertToAsset()` function  will be much less than it actually should be. 

## Code Snippet

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3%2Fcontracts%2Fexternal%2Fproxies%2FBaseERC4626Proxy.sol#L174-L180

## Tool used

Manual Review
VS Code 

## Recommendation

To mitigate this issue, it is recommended to rearrange the operations to perform multiplication before division. This can help prevent the loss of precision.