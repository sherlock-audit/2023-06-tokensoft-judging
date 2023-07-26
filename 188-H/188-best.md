Happy Jetblack Badger

high

# Users can mint more than allocated due implicit conversions

## Summary
The function ```_initializeDistributionRecord``` has a bug which can cause excess mints, when passing in a ```totalAmount``` into the function, it is converted before the check which could be exploited for a massive mint
## Vulnerability Detail
https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/Distributor.sol#L47-L59
As seen the ```_totalAmount``` is converted to a uint120 value, before it is checked:
```solidity
   require(totalAmount <= type(uint120).max, 'Distributor: totalAmount > type(uint120).max');

```
However this check is invalid as a uint120 value can never exceed the ```  type(uint120).max``` .
However if we look at the AdvancedDistributor.sol, we see that a mint happens when the function is called. 
```solidity
  function _initializeDistributionRecord(
    address beneficiary,
    uint256 totalAmount
  ) internal virtual override {
    super._initializeDistributionRecord(beneficiary, totalAmount);

    // add voting power through ERC20Votes extension
    _mint(beneficiary, tokensToVotes(totalAmount));
  }
```
But the amount minted is the original uint256 totalAmount passed in
## Impact
Users can mint more than allocated by Passing in a large value
## Code Snippet

## Tool used

Manual Review

## Recommendation
Check the _totalAmount before conversion