# The Standard
## Contest Summary

Code under review: [2023-12-revert-lend](https://github.com/Cyfrin/2023-12-the-standard) (609 nSLOC)

Contest Page: [the-standard-contest](https://www.codehawks.com/contests/clql6lvyu0001mnje1xpqcuvl)

# Findings
# [M-1] Missing deadline check allow pending transactions to be maliciously executed
### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L214-#L231

## Summary
`SmartVaultV3#swap()` function does not allow users to submit a deadline for their actions which execute swaps on Uniswap. This missing feature enables pending transactions to be maliciously executed at a later point.

## Vulnerability Details
`SmartVaultV3#swap()` function is using `block.timestamp` as deadline which can be exploited by a malicious miner.
```solidity
    function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount) external onlyOwner {
        uint256 swapFee = _amount * ISmartVaultManagerV3(manager).swapFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
        address inToken = getSwapAddressFor(_inToken);
        uint256 minimumAmountOut = calculateMinimumAmountOut(_inToken, _outToken, _amount);
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
                tokenIn: inToken,
                tokenOut: getSwapAddressFor(_outToken),
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,      // <----
                amountIn: _amount - swapFee,
                amountOutMinimum: minimumAmountOut,
                sqrtPriceLimitX96: 0
            });
        inToken == ISmartVaultManagerV3(manager).weth() ?
            executeNativeSwapAndFee(params, swapFee) :
            executeERC20SwapAndFee(params, swapFee);
    }
```
## Impact
Swap can be maliciously executed later, user can face up with the loss when the value of token change. In the worst scenario, vault can be liquidated because of the swap.

## Tools Used
Manual review.

## Recommendations
User should be able to set the deadline.

# [M-2] `costInEuros` calculation will incur precision loss due to division before multiplication

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L220-L221

## Summary
In LiquidationPool::distributeAssets(), `costInEuros` calculation result will be rounded down due to division before multiplication.

## Vulnerability Details
Since exchange rates are involved, round downs will always occur. Here's [code](https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L220-L221) for quick reference.

```solidity
uint256 costInEuros = _portion * 10 ** (18 - asset.token.dec) * uint256(assetPriceUsd) / uint256(priceEurUsd)
    * _hundredPC / _collateralRate;
```
## Impact
These little losses will pile up in proportion the number of users and longevity of the contract.

## Tools Used
Manual Review

## Recommendations
Change the code to this such that multiplication is done first before division. 

```diff
-- uint256 costInEuros = _portion * 10 ** (18 - asset.token.dec) * uint256(assetPriceUsd) / uint256(priceEurUsd)
--    * _hundredPC / _collateralRate;

++ uint256 costInEuros = (_portion * 10 ** (18 - asset.token.dec) * uint256(assetPriceUsd) * _hundredPC) / (uint256(priceEurUsd) * _collateralRate);
```

or this 

```diff
-- uint256 costInEuros = _portion * 10 ** (18 - asset.token.dec) * uint256(assetPriceUsd) / uint256(priceEurUsd)
--    * _hundredPC / _collateralRate;

++ uint256 costInEuros = _portion * 10 ** (18 - asset.token.dec) * uint256(assetPriceUsd) * _hundredPC / uint256(priceEurUsd) / _collateralRate;
```

# [M-3] Incorrect value returned by position() function

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPool.sol#L88

## Summary
The [position()](https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPool.sol#L88) function returns incorrect value for a user's EUROs holdings when balance of `LiquidationPoolManager.sol` contract is greater than zero. It forgets to apply the `_poolFeePercentage`.

## Vulnerability Details
The function is designed to let the caller know of their current EUROs position, including the yet-to-come fee still sitting in the `LiquidationPoolManager` contract. However, it incorrectly considers the full balance of `LiquidationPoolManager` instead of considering only 50% of it (the current `_poolFeePercentage` is 50%). This gives a wrong view to the caller.

```js
    function position(address _holder) external view returns(Position memory _position, Reward[] memory _rewards) {
        _position = positions[_holder];
        (uint256 _pendingTST, uint256 _pendingEUROs) = holderPendingStakes(_holder);
        _position.EUROs += _pendingEUROs;
        _position.TST += _pendingTST;
@---->  if (_position.TST > 0) _position.EUROs += IERC20(EUROs).balanceOf(manager) * _position.TST / getTstTotal();
        _rewards = findRewards(_holder);
    }
```

## PoC
Apply following patch to update `test/liquidationPoolManager.js` and run via `npx hardhat test --grep 'incorrect position function'`. The test will fail since it returns `201` instead of the correct value of `101`:

```diff
diff --git a/test/liquidationPoolManager.js b/test/liquidationPoolManager_position.js
index 7e0f5c2..03ef4f1 100644
--- a/test/liquidationPoolManager.js
+++ b/test/liquidationPoolManager_position.js
@@ -426,5 +426,26 @@ describe('LiquidationPoolManager', async () => {
       expect(rewardAmountForAsset(_rewards, 'WBTC')).to.equal(wbtcCollateral);
       expect(rewardAmountForAsset(_rewards, 'USDC')).to.equal(usdcCollateral);
     });
+    
+    it('incorrect position function', async () => {
+      const tstStake1 = 1;
+      const eurosStake1 = 1;
+      await TST.mint(holder1.address, tstStake1);
+      await EUROs.mint(holder1.address, eurosStake1);
+      await TST.connect(holder1).approve(LiquidationPool.address, tstStake1);
+      await EUROs.connect(holder1).approve(LiquidationPool.address, eurosStake1);
+      let { _rewards, _position } = await LiquidationPool.position(holder1.address);
+      expect(_position.EUROs).to.equal(0);
+      await LiquidationPool.connect(holder1).increasePosition(tstStake1, eurosStake1);
+      ({ _rewards, _position } = await LiquidationPool.position(holder1.address));
+      expect(_position.EUROs).to.equal(eurosStake1);
+      
+      // "mint some fee"
+      await EUROs.mint(LiquidationPoolManager.address, 200);
+      
+      ({ _rewards, _position } = await LiquidationPool.position(holder1.address));
+      // @audit : expected = existing_EUROs + 50% of fee = 1 + 50% of 200 = 101
+      expect(_position.EUROs).to.equal(101, "Incorrect position calculation");
+    });
   });
 });
```

## Impact
This is a `view` function which has so far been only used in the unit tests. However, its design seems to suggest it is supposed to help holders understand their expected holdings, including prospective fees coming their way. "Prospective" because this is still in the `LiquidationPoolManager.sol` contract and `distributeFees()` is yet to be called. As such, this misleads holders about their positions based on which they may be inclined to make further incorrect decisions about withdrawal, staking etc.

## Tools Used
Hardhat

## Recommendations
The `LiquidationPool.sol` contract needs to know the values of `_poolFeePercentage` and `HUNDRED_PC` to calculate this correctly. These can either be initialized in the constructor at deployment time, or need to be passed to the `position()` function:

```diff
-   function position(address _holder) external view returns(Position memory _position, Reward[] memory _rewards) {
+   function position(address _holder, uint256 _poolFeePercentage, uint256 _hundred_pc) external view returns(Position memory _position, Reward[] memory _rewards) {
        _position = positions[_holder];
        (uint256 _pendingTST, uint256 _pendingEUROs) = holderPendingStakes(_holder);
        _position.EUROs += _pendingEUROs;
        _position.TST += _pendingTST;
-       if (_position.TST > 0) _position.EUROs += IERC20(EUROs).balanceOf(manager) * _position.TST / getTstTotal();
+       if (_position.TST > 0) _position.EUROs += IERC20(EUROs).balanceOf(manager) * _position.TST * _poolFeePercentage / (_hundred_pc * getTstTotal());
        _rewards = findRewards(_holder);
    }
```