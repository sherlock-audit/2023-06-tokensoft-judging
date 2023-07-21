Gentle Fuzzy Dalmatian

medium

# CrosschainDistributor: Not paying relayer fee when calling xcall to claim tokens to other domains

## Summary

CrosschainDistributor is not paying relayer fee when calling xcall to claim tokens to other domains. The transaction will not be relayed on target chain to finnalize the claim. User will not receive the claimed tokens unless they bump the transaction fee themself.

## Vulnerability Detail

In `_settleClaim`, the CrosschainDistributor is using xcall to claim tokens to another domain. But relayer fee is not payed.

https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/CrosschainDistributor.sol#L78
```solidity
      id = connext.xcall(            // <------ relayer fee should be payed here
        _recipientDomain, // destination domain
        _recipient, // to
        address(token), // asset
        _recipient, // delegate, only required for self-execution + slippage
        _amount, // amount
        0, // slippage -- assumes no pools on connext
        bytes('') // calldata
      );
```

Without the relayer fee, the transaction will not be relayed. The user will need to bump the relayer fee to finnally settle the claim by following [the instructions here in the connext doc](https://docs.connext.network/integrations/guides/estimating-fees#bumping-relayer-fees).

## Impact

User will not receive their claimed tokens on target chain.

## Code Snippet

https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/CrosschainDistributor.sol#L78

## Tool used

Manual Review

## Recommendation

Help user bump the transaction fee in Satellite.