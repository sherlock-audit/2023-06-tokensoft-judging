Atomic Marmalade Finch

high

# Griefer can block all cross-chain claims

## Summary
A claimer has the option to claim the tokens directly to a `recipient` on a different chain only if they use the `claimBySignature()` function. However, this function can be frontrun and the supplied proof can be supplied to the `claimByMerkleProof()` function instead which will transfer the tokens to the `beneficiary` instead of the `recipient`. **Also**, the tokens will be transferred on the local domain, instead of the desired recipient chain.

## Vulnerability Detail
Since the merkle proofs will work for both function calls, a griefer can ensure that tokens are not sent to the recipient. This is especially impactful because the beneficiary may have chosen to receive the tokens on another chain. The griefing attack can make sure that only the beneficiary can receive the tokens on the same chain as the distributor.

**After consideration, this vulnerability was determined to be of high severity due to the fact that it completely invalidates the purpose of the Tokensoft cross-chain token distribution system.**

## Impact
- Inability to receive tokens on other chains
- Inability to receive tokens on another recipient address

## Code Snippet
https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/CrosschainMerkleDistributor.sol#L93-L104

## Tool used
Manual Review

## Recommendation
Add access control to the `claimByMerkleProof()` function so that only the beneficiary can initiate the claim on the same chain.
