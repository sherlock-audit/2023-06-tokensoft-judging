Cheesy Laurel Beaver

high

# claimBySignature can double spend

## Summary

There is no nonce limit on signatures, and when the beneficiary has votes again, the recipient can continue to double spend with the previous signature.

## Vulnerability Detail

```solidity
  function claimBySignature(
    address _recipient,
    uint32 _recipientDomain,
    address _beneficiary,
    uint32 _beneficiaryDomain,
    uint256 _total,
    bytes calldata _signature,
    bytes32[] calldata _proof
  ) external {
    // Recover the signature by beneficiary
    bytes32 _signed = keccak256(
      abi.encodePacked(_recipient, _recipientDomain, _beneficiary, _beneficiaryDomain, _total)
    );
    address recovered = _recoverSignature(_signed, _signature);
    require(recovered == _beneficiary, '!recovered');

    // Validate the claim
    _verifyMembership(_getLeaf(_beneficiary, _total, _beneficiaryDomain), _proof);
    uint256 claimedAmount = _executeClaim(_beneficiary, _total);

    _settleClaim(_beneficiary, _recipient, _recipientDomain, claimedAmount);
  }
```

There is no nonce for the signature, assuming that the beneficiary authorization recipient can claim A amounts and will burn A token.
After a period of time beneficiary has B token again, and B > A. In this case, the recipient can continue to claim with the previous signature.

## Impact

The recipient can double spend to claim, resulting in beneficiary's loss of funds.

## Code Snippet

- https://github.com/sherlock-audit/2023-06-tokensoft/blob/1f58ddb066ab383c416cfcbf95c9902683506e96/contracts/contracts/claim/abstract/CrosschainMerkleDistributor.sol#L129-L133

## Tool used

Manual Review

## Recommendation

Add a nonce to the signature
