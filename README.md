# Issue H-1: "Votes" balance can be increased indefinitely in multiple contracts 

Source: https://github.com/sherlock-audit/2023-06-tokensoft-judging/issues/41 

## Found by 
0xDanielH, 0xDjango, 0xbranded, 0xlx, AkshaySrivastav, BenRai, Czar102, Musaka, VAD37, Yuki, caventa, dany.armstrong90, jah, jkoppel, magellanXtrachev, ni8mare, p-tsanev, p12473, pengun, r0bert, stopthecap, twicek, y1cunhui
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




## Discussion

**cr-walker**

Great find! We need to preserve the ability to re-initialize distribution records (e.g. if a merkle root changes), so I believe something like this is the best fix:

```
  function _initializeDistributionRecord(
    address beneficiary,
    uint256 totalAmount
  ) internal virtual override {
    super._initializeDistributionRecord(beneficiary, totalAmount);

    uint256 currentVotes = balanceOf(beneficiary);
    uint256 newVotes = tokensToVotes(totalAmount);

    if (currentVotes > newVotes) {
      // reduce voting power through ERC20Votes extension
      _burn(beneficiary, currentVotes - newVotes);
    } else if (currentVotes < newVotes) {
      // increase voting power through ERC20Votes extension
      _mint(beneficiary, newVotes - currentVotes);
    }
  }
  ```

**cr-walker**

Fixed: https://github.com/SoftDAO/contracts/pull/9

# Issue H-2: Users can mint more than allocated due implicit conversions 

Source: https://github.com/sherlock-audit/2023-06-tokensoft-judging/issues/188 

## Found by 
0xbranded, ADM, BenRai, Czar102, Mlome, Musaka, Polaris\_tow, SovaSlava, auditsea, blackhole, jah, josephdara, kutugu, p-tsanev, p12473, panprog, pep7siup, shogoki, y1cunhui, ziyou-
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



## Discussion

**cr-walker**

Looks valid! Great find. It would take an unusual token (eg 100+ decimals) to hit this in practice but it's definitely an error in the logic.

**cr-walker**

Fixed by https://github.com/SoftDAO/contracts/pull/13

# Issue M-1: After InitializeDistributionRecord buying tokens again will DOS claim airdrop in PriceTierVestingSale_2_0 

Source: https://github.com/sherlock-audit/2023-06-tokensoft-judging/issues/14 

## Found by 
Musaka, Oxhunter526, dany.armstrong90, kutugu
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




## Discussion

**cr-walker**

I agree with the high severity issue mentioned (https://github.com/sherlock-audit/2023-06-tokensoft-judging/issues/41) but this issue is solved by the fact that sales always conclude before distributions open (usually by a couple months). I can't see any reference to that in the code so will defer to judges on whether this qualifies as a valid issue.

**Shogoki**

>. I can't see any reference to that in the code so will defer to judges on whether this qualifies as a valid issue.

I intend to leave this a valid Med, as Watsons could not know of this behaviour.


**LayneHaber**

> I intend to leave this a valid Med, as Watsons could not know of this behaviour.

agree

# Issue M-2: setVoteFactor() does not change existing supply of votes. As a result, some may be unable to withdraw. 

Source: https://github.com/sherlock-audit/2023-06-tokensoft-judging/issues/55 

## Found by 
0xDjango, 0xbranded, AkshaySrivastav, BenRai, Czar102, Yuki, auditsea, caventa, jkoppel, mau, p12473, pengun, r0bert
## Summary

`AdvancedDistributor.setVoteFactor()` does not change existing supply of vote tokens. If it is called before all distribution records have been initialized, there will be a skew between those who initialized before and those who initialized after. Further, if it is increased, those who initialized before will not have enough vote tokens to withdraw.

## Vulnerability Detail

Increase scenario (very bad):

1. Owner makes airdrop and sets vote factor to 1.  Many people get a claim to 1000 airdrop tokens.
2. People initialize their distribution records.  They are minted 1000 vote tokens.
3. Owner sets vote factor to 2
4. The airdrop tokens vest
5. No-one who already initialized their distribution record can withdraw anything because they don't have enough vote tokens. (Vote tokens are burned when executing a claim.)

Decrease scenario (less bad):

1. Owner makes an airdrop for 1000 people managed by a CrosschainMerkleDistribution and sets vote factor to 1000
8. User Speedy Gonzalez calls initializeDistributionRecord() for himself
9. Owner decides to change the vote factor to 1 instead
10. All other users only get 1 voting token, but Speedy still has 1000

## Impact

### Increase scenario

If the vote factor is increased after deploying the contract, some people will not be able to withdraw, period.

It is still possible, however, for the owner of the contract to sweep the contract and manually give people their airdrop.

### Decrease scenario

Cannot change vote factor after deploying contract without skewing existing votes.

Note there is no other mechanism to mint or burn vote tokens to correct this.

There is no code that currently uses voting, so this is potentially of no consequence.

However, presumably the voting functionality exists for a reason, and will be used by other code. In particular, the implementation of adjust() takes care to preserve people's number of voting tokens. As the distributor contracts are not upgradeable, this means no fair elections can be run atop airdrops deployed with the current code after setVoteFactor is called.

## Code Snippet

https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/AdvancedDistributor.sol#L181

```solidity
function setVoteFactor(uint256 _voteFactor) external onlyOwner {
    voteFactor = _voteFactor;
    emit SetVoteFactor(voteFactor);
  }
```

setVoteFactor does not change supply

https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/AdvancedDistributor.sol#L77

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

Voting tokens are minted at distribution record initialization time. 

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

`tokensToVotes` uses the current voteFactor. If it has increased since someone's vote tokens were minted, they will not have enough tokens to burn, and so `executeClaim` will revert.

## Tool used

Manual Review

## Recommendation

Do not use separate voting tokens for votes; just use the amount of unclaimed token



## Discussion

**cr-walker**

This is a valid issue.

The admin functions like `setVoteFactor()` should only be called by admins who know what they are doing, but I agree that this would result in unexpected behavior like locking funds due to burning too many vote tokens.

Proposed solution:
* The reason we are using an internal votes token is to allow delegation, but I agree that using the unclaimed token would be simpler and generally better
* we will either remove the `setVoteFactor` method or allow users to reset their voting power, e.g. using something like the solution to: https://github.com/sherlock-audit/2023-06-tokensoft-judging/issues/188

**cr-walker**

Fixed by https://github.com/SoftDAO/contracts/pull/10

# Issue M-3: Because of rounding issues, users may not be able to withdraw airdrop tokens if their claim has been adjust()'ed upwards 

Source: https://github.com/sherlock-audit/2023-06-tokensoft-judging/issues/56 

## Found by 
jkoppel, xAriextz
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



## Discussion

**cr-walker**

Good catch.

Solution: we'll burn the minimum of the expected quantity and current balance to get around these rounding issues.

**cr-walker**

Fixed by https://github.com/sherlock-audit/2023-06-tokensoft-judging/issues/56

# Issue M-4: `Satellite.initiateClaim` encodes the cross chain txn data incorrectly 

Source: https://github.com/sherlock-audit/2023-06-tokensoft-judging/issues/130 

## Found by 
3docSec, AkshaySrivastav, Mohammadreza, kutugu
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



## Discussion

**LayneHaber**

Valid!

**LayneHaber**

fix: https://github.com/SoftDAO/contracts/pull/7

# Issue M-5: `SafeERC20.safeApprove` reverts for changing existing approvals 

Source: https://github.com/sherlock-audit/2023-06-tokensoft-judging/issues/141 

## Found by 
AkshaySrivastav, Juntao, Musaka, auditsea, blackhole, circlelooper, jah, kutugu, ni8mare
## Summary
`SafeERC20.safeApprove` reverts when a non-zero approval is changed to a non-zero approval. The `CrosschainDistributor._setTotal` function tries to change an existing approval to a non-zero value which will revert.

## Vulnerability Detail
The safeApprove function has explicit warning:
```solidity
        // safeApprove should only be called when setting an initial allowance,
        // or when resetting it to zero. To increase and decrease it, use
        // 'safeIncreaseAllowance' and 'safeDecreaseAllowance'
```
But still the `_setTotal` use it to change approval amount:
```solidity
  function _allowConnext(uint256 amount) internal {
    token.safeApprove(address(connext), amount);
  }

  /** Reset Connext allowance when total is updated */
  function _setTotal(uint256 _total) internal virtual override onlyOwner {
    super._setTotal(_total);
    _allowConnext(total - claimed);
  }
```

## Impact
Due to this bug all calls to `setTotal` function of `CrosschainContinuousVestingMerkle` and `CrosschainTrancheVestingMerkle` will get reverted.

Tokensoft airdrop protocol is meant to be used by other protocols and the ability to change `total` parameter is an intended offering. This feature will be important for those external protocols due to the different nature & requirement of every airdrop. But this feature will not be usable by airdrop owners due to the incorrect code implementation.

## Code Snippet
https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/CrosschainDistributor.sol#L35-L45

## Tool used

Manual Review

## Recommendation
Consider using 'safeIncreaseAllowance' and 'safeDecreaseAllowance' instead of `safeApprove` in `_setTotal`.




## Discussion

**cr-walker**

Good catch, updating the code to simply set approval to zero first and then reset approval. I don't think any of the reentrancy attacks that `safeApprove()` is worried about are relevant here (neither the owner nor the connect protocol are going to use this to rug the contract, they both have much more direct ways to take tokens if malicious!)

**cr-walker**

Fixed by https://github.com/SoftDAO/contracts/pull/12

# Issue M-6: CrosschainDistributor: Not paying relayer fee when calling xcall to claim tokens to other domains 

Source: https://github.com/sherlock-audit/2023-06-tokensoft-judging/issues/143 

## Found by 
0xhacksmithh, AkshaySrivastav, Czar102, GREY-HAWK-REACH, Vagner, jkoppel, kutugu, n33k, pengun, pep7siup, qbs, r0bert, tsvetanovv
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



## Discussion

**cr-walker**

Hmm, this would be a valid issue in general but Connext is paying for relayer fees in this case, i.e. a zero fee cross-chain transaction is valid in this case (and on https://docs.connext.network/developers/guides/estimating-fees). @LayneHaber any thoughts on validity?

**cr-walker**

I am marking this as valid since auditors would not know the plan to have Connext pay this fee in this case and we'll change this anyway (it seems strictly better to make the function payable and pass along message value like this:

```
function _settleClaim(
    address _beneficiary,
    address _recipient,
    uint32 _recipientDomain,
    uint256 _amount
  ) internal virtual {
    bytes32 id;
    if (_recipientDomain == 0 || _recipientDomain == domain) {
      token.safeTransfer(_recipient, _amount);
    } else {
      id = connext.xcall{value: msg.value}(
        _recipientDomain, // destination domain
        _recipient, // to
        address(token), // asset
        _recipient, // delegate, only required for self-execution + slippage
        _amount, // amount
        0, // slippage -- assumes no pools on connext
        bytes('') // calldata
      );
    }
    emit CrosschainClaim(id, _beneficiary, _recipient, _recipientDomain, _amount);
  }
  ```

**LayneHaber**

Agree that it is strictly better to have the fees be an option on the contract itself. We can always pass in `0` and the `xcall` will not fail, which works for our distribution but likely not for others.

**Shogoki**

Is there a hint to this behaviour in the connext docs to this behaviour?

**LayneHaber**

docs on the relayer fee behavior can be found [here](https://docs.connext.network/developers/guides/estimating-fees#bumping-relayer-fees) -- this specifically outlines bumping fees, but that implies if the fee is low enough the `xcall` goes through, it's just not processed on the destination chain.

there is no documentation on fee-sponsoring though, and agree that we should make this payable!

**LayneHaber**

Fixed: https://github.com/SoftDAO/contracts/pull/8

