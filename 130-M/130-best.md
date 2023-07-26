Harsh Pewter Peacock

medium

# `Satellite.initiateClaim` encodes the cross chain txn data incorrectly

## Summary
The `initiateClaim` function encodes the data for `connext.xcall` using `abi.encodePacked`, this data is then tried to be decoded in `CrosschainMerkleDistributor.xReceive` function.

Data encoded using `abi.encodePacked` cannot be decoded using `abi.decode` and results in `xReceive` txn getting reverted.

## Vulnerability Detail
The initiateClaim function looks like this:
```solidity
  function initiateClaim(uint256 total, bytes32[] calldata proof) public payable {
    ...
    bytes32 transferId = connext.xcall{ value: msg.value }(
      ...
      abi.encodePacked(msg.sender, _domain, total, proof) // data
    );
    ...
  }
```
`CrosschainMerkleDistributor.xReceive`
```solidity
  function xReceive(
    ...
    bytes calldata _callData
  ) external onlyConnext returns (bytes memory) {
    // Decode the data
    (address beneficiary, uint32 beneficiaryDomain, uint256 totalAmount, bytes32[] memory proof) = abi
      .decode(_callData, (address, uint32, uint256, bytes32[]));
    ...
  }
```
As per Solidity [docs](https://docs.soliditylang.org/en/v0.8.16/abi-spec.html#non-standard-packed-mode)
> In encodePacked, types shorter than 32 bytes are concatenated directly, without padding or sign extension

Hence the data encoded using this non-standard encoding cannot be decoded using `abi.decode`.

## Impact
All the `CrosschainMerkleDistributor.xReceive` calls coming from Satellite via connext will get reverted. 

Satellite is a key contract of Tokensoft cross chain airdrop protocol which facilitates smart-contract wallets in claiming their airdrop tokens. Due to this bug those smart contract recipients won't be able to claim their airdrop tokens.

## Code Snippet
https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/Satellite.sol#L95

## Tool used

Manual Review

## Recommendation
Consider using `abi.encode` instead of `abi.encodePacked`. 