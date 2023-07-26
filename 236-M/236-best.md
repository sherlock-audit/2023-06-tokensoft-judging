Gentle Fuzzy Dalmatian

high

# If the same user address have claimables on different chains, he will lose tokens

## Summary

`records` in Distributor.sol is a mapping from address to claim records. If one address have claimables on different chains, the records could overlap, resulting in user loss.

## Vulnerability Detail

We can see that in Distributor.sol that `records` is a mapping from address to claim records.

```solidity
  function _initializeDistributionRecord(
    address beneficiary,
    uint256 _totalAmount
  ) internal virtual {
    uint120 totalAmount = uint120(_totalAmount);

    // Checks
    require(totalAmount <= type(uint120).max, 'Distributor: totalAmount > type(uint120).max');

    // Effects - note that the existing claimed quantity is re-used during re-initialization
    records[beneficiary] = DistributionRecord(true, totalAmount, records[beneficiary].claimed);
    emit InitializeDistributionRecord(beneficiary, totalAmount);
  }

  function _executeClaim(
    address beneficiary,
    uint256 _totalAmount
  ) internal virtual returns (uint256) {
    uint120 totalAmount = uint120(_totalAmount);

    // effects
    if (records[beneficiary].total != totalAmount) {
      // re-initialize if the total has been updated
      _initializeDistributionRecord(beneficiary, totalAmount);
    }
    
    uint120 claimableAmount = uint120(getClaimableAmount(beneficiary));
    require(claimableAmount > 0, 'Distributor: no more tokens claimable right now');

    records[beneficiary].claimed += claimableAmount;
    claimed += claimableAmount;

    return claimableAmount;
  }
```

The same address will share a same record with a same `claimed` field. This field is increasing only, so user will lossing he claimables because of the overlapping.

## Impact

User lose claimables if he has claimables on different chains.

## Code Snippet

https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/Distributor.sol#L47-L85

## Tool used

Manual Review

## Recommendation

Add a mapping key of domain in records in CrosschainDistributor.