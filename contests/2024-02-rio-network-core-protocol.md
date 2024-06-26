# Rio Network Core Protocol
## Contest Summary

Code under review: [2024-02-rio-network-core-protocol](https://github.com/sherlock-audit/2024-02-rio-network-core-protocol-judging/issues) (2,453 nSLOC)

Contest Page: [rio-network-core-protocol](https://audits.sherlock.xyz/contests/176)

## [H-1] Settling with Eigenlayer does not update Epoch count, causing the protocol to lock all the funds.
### Summary
To facilitate deposits into EigenLayer and to process 1-day withdrawals from the Rio LRT, there is an automated process that is run called the Deposit Pool Rebalancer. This function is called via the Coordinator.

If the amount in the Deposit Pool can not fund the sum of the user's withdrawal requests, then the full withdrawal will be queued for redemption from EigenLayer. The withdrawal(s) will have the full 7-day EigenLayer withdrawal delay.

### Vulnerability Detail
The rebalance function is critical as it initiates the restaking process into Eigenlayer and facilitates withdrawal from both the deposit pool and Eigenlayer. Within the function, if the deposit pool does not have sufficient assets to support withdrawal, hence the RioLRTWithdrawalQueue::settleEpochFromEigenLayer function is called. It fails to increase the epoch count and mark the epoch as settled. After 24 hours, the rebalance function will be triggered again and will always revert failing the restaking process into Eigenlayer and facilitates withdrawals.

### Impact
Restaked assets will be locked causing stakers to lose their assets.
The protocol is unable to restake the underlying assets into Eigenlayer.
### Code Snippet
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/main/rio-sherlock-audit/contracts/restaking/RioLRTWithdrawalQueue.sol#L166
```solidity### 
// Function is called when depositPool has sufficient funds to support withdrawal
 function settleCurrentEpoch(address asset, uint256 assetsReceived, uint256 shareValueOfAssetsReceived) {
...
        // This is included
        epochWithdrawals.settled = true;
        currentEpochsByAsset[asset] += 1;
}
```
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/main/rio-sherlock-audit/contracts/restaking/RioLRTWithdrawalQueue.sol#L216
```solidity
// Function is called when depositPool does not have sufficient funds to support withdrawal
    function settleEpochFromEigenLayer(
        address asset,
        uint256 epoch,
        IDelegationManager.Withdrawal[] calldata queuedWithdrawals,
        uint256[] calldata middlewareTimesIndexes
    ) external {
      ...
        epochWithdrawals.settled = true;
      //  Function missing increment of epoch
```
```solidity
  function test_attack() public {
        uint256 initialTotalSupply = reLST.token.totalSupply();

        uint8 operatorId = addOperatorDelegator(reLST.operatorRegistry, address(reLST.rewardDistributor));
        address operatorDelegator = reLST.operatorRegistry.getOperatorDetails(operatorId).delegator;

        uint256 amount = 8e18;

        // Deposit cbETH and rebalance to move all tokens to EigenLayer.
        cbETH.approve(address(reLST.coordinator), type(uint256).max);
        uint256 restakingTokensOut = reLST.coordinator.deposit(CBETH_ADDRESS, amount);

        vm.prank(EOA, EOA);
        reLST.coordinator.rebalance(CBETH_ADDRESS);

        // Request a withdrawal for the tokens from EigenLayer and rebalance.
        reLST.coordinator.requestWithdrawal(CBETH_ADDRESS, restakingTokensOut);
        skip(reLST.coordinator.rebalanceDelay());

        vm.prank(EOA, EOA);
        reLST.coordinator.rebalance(CBETH_ADDRESS);

        // Ensure no reLST has been burned yet.
        assertEq(reLST.token.totalSupply(), restakingTokensOut + initialTotalSupply);

        // Settle the withdrawal epoch.
        uint256 withdrawalEpoch = reLST.withdrawalQueue.getCurrentEpoch(CBETH_ADDRESS);
        IDelegationManager.Withdrawal[] memory withdrawals = new IDelegationManager.Withdrawal[](1);
        withdrawals[0] = IDelegationManager.Withdrawal({
            staker: operatorDelegator,
            delegatedTo: address(1),
            withdrawer: address(reLST.withdrawalQueue),
            nonce: 0,
            startBlock: 1,
            strategies: CBETH_STRATEGY.toArray(),
            shares: amount.toArray()
        });
        reLST.withdrawalQueue.settleEpochFromEigenLayer(CBETH_ADDRESS, withdrawalEpoch, withdrawals, new uint256[](1));

        // Assert epoch summary details.
        IRioLRTWithdrawalQueue.EpochWithdrawalSummary memory epochSummary =
            reLST.withdrawalQueue.getEpochWithdrawalSummary(CBETH_ADDRESS, withdrawalEpoch);
        assertTrue(epochSummary.settled);
        assertEq(epochSummary.assetsReceived, amount);
        assertEq(epochSummary.shareValueOfAssetsReceived, amount); // Share value is 1:1 initially.

        // Claim and assert withdrawal.
        uint256 balanceBefore = cbETH.balanceOf(address(this));
        uint256 cbETHWithdrawn = reLST.withdrawalQueue.claimWithdrawalsForEpoch(
            IRioLRTWithdrawalQueue.ClaimRequest({asset: CBETH_ADDRESS, epoch: withdrawalEpoch})
        );
        IRioLRTWithdrawalQueue.UserWithdrawalSummary memory userSummary =
            reLST.withdrawalQueue.getUserWithdrawalSummary(CBETH_ADDRESS, withdrawalEpoch, address(this));

        assertTrue(userSummary.claimed);
        assertEq(cbETHWithdrawn, amount);
        assertEq(cbETH.balanceOf(address(this)) - balanceBefore, amount);

        ////////////Second Rebalance//////////////
        skip(reETH.coordinator.rebalanceDelay());
        uint256 secondinitialTotalSupply = reLST.token.totalSupply();


        // Another  deposit cbETH and rebalance to move all tokens to EigenLayer.
        cbETH.approve(address(reLST.coordinator), type(uint256).max);
        uint256 secondrestakingTokensOut = reLST.coordinator.deposit(CBETH_ADDRESS, amount);

        vm.prank(EOA, EOA);
        vm.expectRevert(IRioLRTWithdrawalQueue.EPOCH_ALREADY_SETTLED.selector);
        // Unable to be called, Hence funds locked forever
        reLST.coordinator.rebalance(CBETH_ADDRESS);
```
### Tool used
Manual Review

### Recommendation
Include the missing currentEpochsByAsset[asset] += 1;  at the end of the RioLRTWithdrawalQueue::settleEpochFromEigenLayer function.

