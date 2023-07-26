Harsh Pewter Peacock

medium

# `SafeERC20.safeApprove` reverts for changing existing approvals

## Summary
`SafeERC20.safeApprove` reverts when a non-zero approval is changed to a non-zero approval. The `CrosschainDistributor._setTotal` function tries to change an existing approval to a non-zero value which will revert.

## Vulnerability Detail
The safeApprove function has explicit warning:
```solidity
        // safeApprove should only be called when setting an initial allowance,
        // or when resetting it to zero. To increase and decrease it, use
        // 'safeIncreaseAllowance' and 'safeDecreaseAllowance'
```
But still the `_setTotal` use it to change approval amount:
```solidity
  function _allowConnext(uint256 amount) internal {
    token.safeApprove(address(connext), amount);
  }

  /** Reset Connext allowance when total is updated */
  function _setTotal(uint256 _total) internal virtual override onlyOwner {
    super._setTotal(_total);
    _allowConnext(total - claimed);
  }
```

## Impact
Due to this bug all calls to `setTotal` function of `CrosschainContinuousVestingMerkle` and `CrosschainTrancheVestingMerkle` will get reverted.

Tokensoft airdrop protocol is meant to be used by other protocols and the ability to change `total` parameter is an intended offering. This feature will be important for those external protocols due to the different nature & requirement of every airdrop. But this feature will not be usable by airdrop owners due to the incorrect code implementation.

## Code Snippet
https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/CrosschainDistributor.sol#L35-L45

## Tool used

Manual Review

## Recommendation
Consider using 'safeIncreaseAllowance' and 'safeDecreaseAllowance' instead of `safeApprove` in `_setTotal`.
