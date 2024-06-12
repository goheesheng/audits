# Open Dollar
## Contest Summary

Code under review: [2024-03-open-dollar](https://github.com/code-423n4/2023-10-open-dollar)) (998 nSLOC)

Contest Page: [revert-open-dollar](https://code4rena.com/audits/2023-10-open-dollar)

# Findings

## [M-1] Unprotected initialize Implementation

### Lines of code
https://github.com/open-dollar/od-contracts/blob/f4f0246bb26277249c1d5afe6201d4d9096e52e6/src/contracts/proxies/ODSafeManager.sol#L64

### Vulnerability details
### Impact
Not being able to be the SafeManager and NFTRenderer of the vault, allowing malicious actors to take over the vault and break the protocol.

P.S I consider this as high risk because this exploit is a multiple of DOS, Frontrunning and Access Control which can lead to protocol failure.

### Proof of Concept
It should be avoided that the implementation of contracts can be initialized by third parties. This can be the case if the initializeManager & initializeRenderer function is unprotected. Since the implementation contract is not meant to be used directly without the SafeManager it is recommended to protect the initialization method of the implementation using access control.

The pragmatic way of exploiting such case is by frontrunning:

1) Vault is deployed followed by ODSafeManager.
2) Malcious Validator noticed that the vault is deployed.
3) Malicious Validator frontrun the transaction by calling initializeManager or initializeRenderer
4) When the ODSafeManager or NFTRenderer is deployed, it tries to call the initializeManager or initializeRenderer
5) Both contracts can never call the functions since it has already been called.
6) You will notice that the initializeManager & initializeManager functions do check if the function has been called. Since we know that the default value of the address variable is zero-state this means that once we called the function, it can only be called once.
```solidity
contracts/proxies/ODSafeManager.sol

  constructor(address _safeEngine, address _vault721) {
    safeEngine = _safeEngine.assertNonNull();
    vault721 = IVault721(_vault721);
    vault721.initializeManager();
  }

https://github.com/open-dollar/od-contracts/blob/f4f0246bb26277249c1d5afe6201d4d9096e52e6/src/contracts/proxies/ODSafeManager.sol#L64C1-L68C4
```
```solidity
contracts/proxies/NFTRenderer.sol

  constructor(address _vault721, address oracleRelayer, address taxCollector, address collateralJoinFactory) {
    vault721 = IVault721(_vault721);
    vault721.initializeRenderer();
    _safeManager = IODSafeManager(vault721.safeManager());
    _safeEngine = ISAFEEngine(_safeManager.safeEngine());
    _oracleRelayer = IOracleRelayer(oracleRelayer);
    _taxCollector = ITaxCollector(taxCollector);
    _collateralJoinFactory = ICollateralJoinFactory(collateralJoinFactory);
  }

https://github.com/open-dollar/od-contracts/blob/f4f0246bb26277249c1d5afe6201d4d9096e52e6/src/contracts/proxies/NFTRenderer.sol#L35C1-L43C4
```solidity
contracts/proxies/Vault721.sol

  function initializeManager() external {
    if (address(safeManager) == address(0)) _setSafeManager(msg.sender);
  }

  function initializeRenderer() external {
    if (address(nftRenderer) == address(0)) _setNftRenderer(msg.sender);
  }

https://github.com/open-dollar/od-contracts/blob/f4f0246bb26277249c1d5afe6201d4d9096e52e6/src/contracts/proxies/Vault721.sol#L56C1-L58C4
```
### Tools Used
Manual Review

### Recommended Mitigation Steps
Protect initialization methods from being called by unauthorized parties. Create RBAC only allowing Vault and NFTRenderer contracts to only access the specified functions.

### Assessed type
Access Control
