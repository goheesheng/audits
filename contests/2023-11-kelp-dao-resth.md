# Kelp DAO | rsETH
## Contest Summary

Code under review: [2023-11-kelp-dao-rseth(https://github.com/code-423n4/2023-11-kelp-dao-rseth) (3000 nSLOC)

Contest Page: [kelp-dao-rseth](https://code4rena.com/audits/2023-11-kelp-dao-rseth)


# Findings

## [M-1] Max Approval
### Lines of code
https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/NodeDelegator.sol#L38

### Vulnerability details
### Impact
For safety, the approval should not be set to the max value, especially if the amount that the contract may use is already known in this call. If the contract is compromised, it can steal all the tokens.

### Proof of Concept
```solidity
    /// @notice Approves the maximum amount of an asset to the eigen strategy manager
    /// @dev only supported assets can be deposited and only called by the LRT manager
    /// @param asset the asset to deposit
    function maxApproveToEigenStrategyManager(address asset)
        external
        override
        onlySupportedAsset(asset)
        onlyLRTManager
    {
        address eigenlayerStrategyManagerAddress = lrtConfig.getContract(LRTConstants.EIGEN_STRATEGY_MANAGER);
        IERC20(asset).approve(eigenlayerStrategyManagerAddress, type(uint256).max);
    }
```
### Tools Used
Manual Review

### Recommended Mitigation Steps
Only approve appropriate amounts before transfer.

### Assessed type
ERC20
