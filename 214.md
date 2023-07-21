Soaring Ebony Moth

medium

# Anyone can reduce the native coin balance of Distributor to 0

## Summary
Anyone can reduce the native coin balance of the distributor to 0 by calling the sweepNative() function of Sweepable. This will send the money to a predetermined recipient and will cause denial of service for any functions that rely on having ether/native coins.
## Vulnerability Detail
The Sweepable contract is a contract inherited by AdvancedDistributor. AdvancedDistributor on the other hand is inherited by all other distributors.

Through Sweepable the Owner can send the native coins balance and different erc20 token balances to a predefined recipient.

All of these functions have the onlyOwner modifier, except for **sweepNative()**
```solidity
    function sweepNative() external { 
        uint256 amount = address(this).balance;
        (bool success, ) = recipient.call{value: amount}("");
        require(success, "Transfer failed.");
        emit SweepNative(amount);
    }

    function sweepNative(uint256 amount) external onlyOwner {
        (bool success, ) = recipient.call{value: amount}("");
        require(success, "Transfer failed.");
        emit SweepNative(amount);
    }
```

SweepNative allows any user (including the recipient), to sweep the native balance from the contract and cause all balance-related functionalities to stop working.
## Impact
Insufficient balance of the Distributor contract can cause different forms of Denial of Service, depending on the exact Distributor implementation.
A compromised recipient could steal the funds from the contract for himself.
## Code Snippet
https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/utilities/Sweepable.sol#L34-L39

## Tool used

Manual Review

## Recommendation
Add the onlyOwner modifier to the sweepNative function.