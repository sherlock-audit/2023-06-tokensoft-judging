Gentle Aqua Raven

medium

# Users can force others to claim, reducing their voting share

## Summary

Because the varying claim functions can generally be called by anyone, anyone can force a user to claim their whole vested share, losing voting tokens in the process. This is particularly hard to fix for `claimBySignature`  because of replays.

## Vulnerability Detail

All claim functions except `claimBySignature` can be called by anyone, reducing their voting share. Here's a scenario:

1. Owner runs an airdrop, giving 1000 users 1000 tokens each
2. All users fully vest
3. All users decide to keep their share unclaimed, deciding their voting power is valuable
4. Owner decides to run an election
5. Right before the close of the election, a malicious user forces all other users to claim their share, leaving himself with 100% of the votes.

## Impact

There is no code that currently uses voting, so potentially none.

However, presumably the voting functionality exists for a reason, and will be used by other code. As the distributor contracts are not upgradeable, this hinders running fair elections atop airdrops deployed with the current code.

Many comments indicate significant care has been taken to keep voting power fair, and to provide bonuses to users who choose not to withdraw.  This vulnerability stymies that.

E.g.:

https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/test/distributor/continuousVestingMerkle.test.ts#L350

```solidity
      // a factor of two is applied to all unclaimed tokens for voting power
```

https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/PriceTierVestingSale_2_0.sol#L38

```solidity
    uint256 _voteFactor, // the factor for voting power in basis points (e.g. 15000 means users have a 50% voting bonus for unclaimed tokens)
```


## Code Snippet

Nearly all claim functions can be called by anyone. E.g.:

https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/BasicDistributor.sol#L46

```solidity
	function claim(address beneficiary) external nonReentrant {
		// effects
		uint256 claimedAmount = super._executeClaim(beneficiary, records[beneficiary].total);
		// interactions
		super._settleClaim(beneficiary, claimedAmount);
	}
```

## Tool used

Manual Review

## Recommendation

Only allow the beneficiary to claim. For cross-chain calls, check originSender or require signature from the beneficiary. Or just abolish the voting.