Custom Chiffon Mantis

high

# "Votes" balance can be increased indefinitely in multiple contracts

## Summary
The "voting power" can be easily manipulated in the following contracts:
- `ContinuousVestingMerkle`
- `PriceTierVestingMerkle`
- `PriceTierVestingSale_2_0`
- `TrancheVestingMerkle`
- `CrosschainMerkleDistributor`
- `CrosschainContinuousVestingMerkle`
- `CrosschainTrancheVestingMerkle`
- All the contracts inheriting from the contracts listed above

This is caused by the public `initializeDistributionRecord()` function that can be recalled multiple times without any kind of access control:
```solidity
  function initializeDistributionRecord(
    uint32 _domain, // the domain of the beneficiary
    address _beneficiary, // the address that will receive tokens
    uint256 _amount, // the total claimable by this beneficiary
    bytes32[] calldata merkleProof
  ) external validMerkleProof(_getLeaf(_beneficiary, _amount, _domain), merkleProof) {
    _initializeDistributionRecord(_beneficiary, _amount);
  }
```

## Vulnerability Detail
The `AdvancedDistributor` abstract contract which inherits from the `ERC20Votes`, `ERC20Permit` and `ERC20` contracts, distributes tokens to beneficiaries with voting-while-vesting and administrative controls. Basically, before the tokens are vested/claimed by a certain group of users, these users can use these ERC20 tokens to vote. These tokens are minted through the `_initializeDistributionRecord()` function:
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
As mentioned in the [Tokensoft Discord channel](https://discord.com/channels/812037309376495636/1130514276570906685/1130577295539707995) these ERC20 tokens minted are used to track an address's unvested token balance, so that other projects can utilize 'voting while vesting'.

A user can simply call as many times as he wishes the `initializeDistributionRecord()` function with a valid merkle proof. With each call, the `totalAmount` of tokens will be minted. Then, the user simply can call `delegate()` and delegate those votes to himself, "recording" the inflated voting power.

## Impact
The issue totally breaks the 'voting while vesting' design. Any DAO/project using these contracts to determine their voting power could be easily manipulated/exploited.

## Code Snippet
- https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/ContinuousVestingMerkle.sol#L43-L53
- https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/PriceTierVestingMerkle.sol#L49-L59
- https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/PriceTierVestingSale_2_0.sol#L91-L95
- https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/TrancheVestingMerkle.sol#L39-L49
- https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/CrosschainMerkleDistributor.sol#L46-L53

## Tool used
Manual Review

## Recommendation
Only allow users to call once the `initializeDistributionRecord()` function. Consider using a mapping to store if the function was called previously or not. Keep also in mind that fully vested and claimed users should not be able to call this function and if they do, the total amount of tokens that should be minted should be 0 or proportional/related to the amount of tokens that they have already claimed.
