Sweet Blood Cod

medium

# Incorrect storage gap in `wfCashBase` and `BaseERC4626Proxy` can lead to data corruption in storage when upgrading

## Summary
Parent/ascendant contracts of all upgradeable contracts must have a uniform storage size of 50 to prevent data corruption during upgrades caused by state variables changing the storage slots they use. This happens when state variables are added or removed and shifts storage down or up. 

## Vulnerability Detail
`wfCashERC4626` is an upgradeable contract that inherits from `wfCashBase`. Being a parent contract, `wfCashBase` uses a [storage gap](https://docs.openzeppelin.com/contracts/4.x/upgradeable#storage_gaps) to protect against upgrades shifting down state variables and causing data corruption since state variables in descendant contracts end up using different storage slots after the upgrade. The size of the gap is 50 by convention and it is advisable to have uniform storage gap across all contracts in the codebase to prevent upgrades changing the storage slots state variables use.

The gap size for `wfCashBase` is set to 45:
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashBase.sol#L235-L240

```solidity
/**
 * @dev This empty reserved space is put in place to allow future versions to add new
 * variables without shifting down storage in the inheritance chain.
 * See https://docs.openzeppelin.com/contracts/4.x/upgradeable#storage_gaps
 */
uint256[45] private __gap;
```

`wfCashBase` has the following state variables declared:

https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashBase.sol#L19-L31
```solidity
    // This is the empirically measured maximum amount of fCash that can be lent. Lending
    // the fCashAmount down to zero is not possible due to rounding errors and fees inside
    // the liquidity curve.
    uint256 internal constant MAX_LEND_LIMIT = 1e9 - 50;
    // Below this absolute number we consider the max fcash to be zero
    uint256 internal constant MIN_FCASH_FLOOR = 50_000;

    /// @notice address to the NotionalV2 system
    INotionalV2 public immutable NotionalV2;
    WETH9 public immutable WETH;

    /// @dev Storage slot for fCash id. Read only and set on initialization
    uint64 private _fCashId;
```

Only one storage slot is used by the contract, which is the one used by `_fCashId`. Constants and immutables do not use storage since their values are embedded in the bytecode of the contract. 

The correct gap size for `wfCashBase` is 49. 

`BaseERC4626Proxy` also has the same issue. It sets a gap size of 40:
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/proxies/BaseERC4626Proxy.sol#L386-L387

```solidity
    // This is here for safety, but inheriting contracts should never declare storage anyway
    uint256[40] __gap;
```

`BaseERC4626Proxy` declares the following state variables:
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/proxies/BaseERC4626Proxy.sol#L43-L68

```solidity
    /// @notice Inherits from Constants.INTERNAL_TOKEN_PRECISION
    uint8 public constant decimals = 8;

    /// @notice Precision for exchangeRate()
    uint256 public constant EXCHANGE_RATE_PRECISION = 1e18;

    /// @notice Address of the notional proxy, proxies only have access to a subset of the methods
    NotionalProxy public immutable NOTIONAL;

    /*** STORAGE SLOTS [SET ONCE ON EACH PROXY] ***/

    /// @notice Will be "[Staked] nToken {Underlying Token}.name()", therefore "USD Coin" will be
    /// "nToken USD Coin" for the regular nToken and "Staked nToken USD Coin" for the staked version.
    string public name;

    /// @notice Will be "[s]n{Underlying Token}.symbol()", therefore "USDC" will be "nUSDC"
    string public symbol;

    /// @notice Currency id that this nToken refers to
    uint16 public currencyId;

    /// @notice Native underlying decimal places
    uint8 public nativeDecimals;

    /// @notice ERC20 underlying token referred to as the "asset" in IERC4626
    address public underlying;
```

In this case, 4 storage slots are used by `BaseERC4626Proxy` because of the following:
1. `bool _initialized` and `bool _initializing` in `Initializable` - use 1 slot since they share 1 32-byte storage slot. These state variables use a total of 16 bits (2 bytes) since booleans use 8 bits.
2. `string public name` - 1 slot
3. `string public symbol` - 1 slot
4. `uint16 public currencyId` and `uint8 public nativeDecimals` and `address public underlying` - use 1 since they share 1 32-byte storage slot. These state variables only use a total of 184 bits.

Note that a state variable will share a storage slot with the next declared state variable if they can fit within one 32-byte slot. The correct gap size for `BaseERC4626Proxy` is 46.

## Impact
The protection provided by storage gaps do not apply for `BaseERC4626Proxy` and `wfCashBase` since they are used incorrectly. It risks upgrades compromising storage for those contracts.

## Code Snippet
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/wrapped-fcash/contracts/wfCashBase.sol#L19-L31
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/main/contracts-v3/contracts/external/proxies/BaseERC4626Proxy.sol#L43-L68

## Tool used
Manual Review

## Recommendation
Consider correcting the gap sizes for `wfCashBase` and `BaseERC4626Proxy` to the following:
1. `wfCashBase` - 49
2. `BaseERC4626Proxy` - 46

Since the child contracts of `wfCashBase` and `BaseERC4626Proxy` do not add their own state variables, it would be safe to upgrade the contracts in production with the changes to the storage gaps in these ascendant contracts as long as no changes are made to the state variables during this upgrade to correct the gap sizes.