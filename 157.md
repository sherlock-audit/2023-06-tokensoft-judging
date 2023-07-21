Keen Mustard Panda

medium

# `_settleClaim` in `CrosschainDistributor` will revert for cross chain claims because `Connext` does not have the allowance for sending tokens

## Summary

When deploying `CrosschainTrancheVestingMerkle.sol` the allowance of `Connext` for the token is set before the token variable itself is set. This means after deploying the distributor, `Connext` does not have any allowance for the token stored in the distributor. This leads to a revert when `_settleClaim` in `CrosschainDistributor` is called for a cross chain claim.


## Vulnerability Detail

When launching the `CrosschainTrancheVestingMerkle` contract the constructors of `CrosschainDistributor` is executed before the constructor of `AdvancedDistributor`. The allowance for `Connext` is set in the constructor of `CrosschainDistributor` using the `token` variable but the `token` variable is set afterwards in the `AdvancedDistributor`. This means that no allowance for `Connext` for the `token` is set during deployment. This leads to a revert if someone tries to use `_settleClaim` in `CrosschainDistributor` for a cross chain claim since `Connext` can not transfare the tokens to itself.

Copy this POC into `CrosschainTrancheVestingMerkle.test.ts`:

```solidity

it.only("No alowance set for Connext", async () => {
    let allowance = await distributor.allowance(
      distributor.address,
      connextMockDestination.address
    );

    console.log("Allowance connext", allowance.toBigInt());

    expect(allowance.toBigInt()).toEqual(0n);
  });   

```

## Impact

Recipients can not execute their cross chain claims 

## Code Snippet

https://github.com/sherlock-audit/2023-06-tokensoft/blob/1f58ddb066ab383c416cfcbf95c9902683506e96/contracts/contracts/claim/CrosschainTrancheVestingMerkle.sol#L18-L30

https://github.com/sherlock-audit/2023-06-tokensoft/blob/1f58ddb066ab383c416cfcbf95c9902683506e96/contracts/contracts/claim/abstract/CrosschainMerkleDistributor.sol#L39-L43

https://github.com/sherlock-audit/2023-06-tokensoft/blob/1f58ddb066ab383c416cfcbf95c9902683506e96/contracts/contracts/claim/abstract/CrosschainDistributor.sol#L25-L29

https://github.com/sherlock-audit/2023-06-tokensoft/blob/1f58ddb066ab383c416cfcbf95c9902683506e96/contracts/contracts/claim/abstract/AdvancedDistributor.sol#L51-L68


## Tool used

Manual Review, Test in repo

## Recommendation

Make sure that the token is set before giving the allowance for the token to `Connext`. This should be doable by changing the order in the constructor and putting the constructor of `TrancheVesting` (that calls the constructor of `AdvancedDistributor` that sets the token) before the constructor of `CrosschainMerkleDistributor`
