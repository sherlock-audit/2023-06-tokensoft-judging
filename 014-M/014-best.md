Cheesy Laurel Beaver

medium

# After InitializeDistributionRecord buying tokens again will DOS claim airdrop in PriceTierVestingSale_2_0

## Summary

There are two issues involved here:
1. InitializeDistributionRecord can be called multiple times by anyone, it seems to be a problem, because users can mint multiple times vote token. The second issue occurs after fixing this issue.
2. InitializeDistributionRecord will mint corresponding votes for the user according to the current purchase quantity, but users can also continue to buy tokens, and getPurchasedAmount at this time more than before, when the user claim airdrop at this time, need to burn more tokens, but user has fewer mint tokens than this, resulting in claim DOS.

## Vulnerability Detail

```solidity
  function getPurchasedAmount(address buyer) public view returns (uint256) {
    /**
    Get the quantity purchased from the sale and convert it to native tokens
  
    Example: if a user buys $1.11 of a FOO token worth $0.50 each, the purchased amount will be 2.22 FOO
    - buyer total: 111000000 ($1.11 with 8 decimals)
    - decimals: 6 (the token being purchased has 6 decimals)
    - price: 50000000 ($0.50 with 8 decimals)

    Calculation: 111000000 * 1000000 / 50000000

    Returns purchased amount: 2220000 (2.22 with 6 decimals)
    */
    return (sale.buyerTotal(buyer) * (10 ** soldTokenDecimals)) / price;
  }

  function initializeDistributionRecord(
    address beneficiary // the address that will receive tokens
  ) external validSaleParticipant(beneficiary) {
    _initializeDistributionRecord(beneficiary, getPurchasedAmount(beneficiary));
  }

  function claim(
    address beneficiary // the address that will receive tokens
  ) external validSaleParticipant(beneficiary) nonReentrant {
    uint256 claimableAmount = getClaimableAmount(beneficiary);
    uint256 purchasedAmount = getPurchasedAmount(beneficiary);

    // effects
    uint256 claimedAmount = super._executeClaim(beneficiary, purchasedAmount);

    // interactions
    super._settleClaim(beneficiary, claimedAmount);
  }
```

getPurchasedAmount depends on the current user's buyerTotal, when the user initializeDistributionRecord and purchase again will cause purchasedAmount overflow, can't claim the airdrop.
The specific process is: buy first -> initializeDistributionRecord(mint) -> buy second -> claim(burn).
The second purchase makes the purchasedAmount greater than the first purchase, that is, when the user claims airdrop, burn > mint, DOS claim.

## Impact

After InitializeDistributionRecord buying tokens again will DOS claim airdrop

## Code Snippet

- https://github.com/sherlock-audit/2023-06-tokensoft/blob/1f58ddb066ab383c416cfcbf95c9902683506e96/contracts/contracts/claim/PriceTierVestingSale_2_0.sol#L91-L108

## Tool used

Manual Review

## Recommendation

Should not be allowed users to buy again, after calling InitializeDistributionRecord
