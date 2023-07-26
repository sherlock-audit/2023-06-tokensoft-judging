Gentle Aqua Raven

medium

# Because of rounding issues, users may not be able to withdraw airdrop tokens if their claim has been adjust()'ed upwards

## Summary

In order for a user to withdraw their claim, they must have enough voting tokens. However, because of rounding issues, if their voting shares are granted in multiple stages, namely by the owner adjust()-ing their share upwards,  they will not have enough.

## Vulnerability Detail

1. Owner creates airdrop and grants a user a claim of 1000 tokens. The voting factor is 5, and the fractionDenominator is set to 10000.
2. User initializes their distribution record. They are minted 1000*5/10000 = 0 voting tokens.
3. Owner adjusts everyone's claim up to 1000. Each user is minted another 1000*5/10000=0 voting tokens.
4. User fully vests
5. User cannot withdraw anything because, in order to withdraw, they must burn 2000*5/10000= 1 voting token.

## Impact

Unless all grants and positive adjust()'s are for exact multiples of fractionDenominator, users will be prevented from withdrawing after an upwards adjustment.

Note that many comments give example values of 10000 for fraction denominator and 15000 for voteFactor. Since the intention is to use `voteFactor`'s which are not multiples of fractionDenonimator, rounding issues will occur.

## Code Snippet

Rounding in tokensToVotes

https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/AdvancedDistributor.sol#L73

```solidity
  function tokensToVotes(uint256 tokenAmount) private view returns (uint256) {
    return (tokenAmount * voteFactor) / fractionDenominator;
  }
```

_inititializeDistributionRecord and adjust() both use  `tokensToVotes` to mint

https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/AdvancedDistributor.sol#L84C1-L85C1

https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/AdvancedDistributor.sol#L126


tokensToVotes is again used to burn when executing a claim

https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/AdvancedDistributor.sol#L87

```solidity

  function _executeClaim(
    address beneficiary,
    uint256 totalAmount
  ) internal virtual override returns (uint256 _claimed) {
    _claimed = super._executeClaim(beneficiary, totalAmount);

    // reduce voting power through ERC20Votes extension
    _burn(beneficiary, tokensToVotes(_claimed));
  }

```

## Tool used

Manual Review

## Recommendation

Base votes on share of unclaimed tokens and not on a separate token.