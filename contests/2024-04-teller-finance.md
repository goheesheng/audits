# Teller Finance

## Contest Summary

Code under review: [2024-04-teller-finance](https://github.com/sherlock-audit/2024-04-teller-finance-judging/issues) (1460 nSLOC)

Contest Page: [Teller-Finance-contest](https://audits.sherlock.xyz/contests/295)


# Findings

## [H-1] burnSharesToWithdrawEarnings causing more asset token withdrawal for users
### Summary

`burnSharesToWithdrawEarnings` is used for lenders to convert shares back into assets.

### Vulnerability Detail
The conversion has a rate utilize poolSharesToken.totalSupply(). The problem with the implementation is that LP tokens are burned before calculating withdrawal, leading to unfair withdrawal amounts.
```solidity
    function burnSharesToWithdrawEarnings(
        uint256 _amountPoolSharesTokens,
        address _recipient
    ) external returns (uint256) {
      
        //@audit-issue
        poolSharesToken.burn(msg.sender, _amountPoolSharesTokens);

        uint256 principalTokenValueToWithdraw = _valueOfUnderlying(
            _amountPoolSharesTokens,
            sharesExchangeRateInverse() //@audit returned lower, causing asset value to be HIGHER
        );
```
Within the burnSharesToWithdrawEarnings() it utilizes the sharesExchangeRate(). Knowing that the totalSupply decreases first from the burn function. This means that the rate for sharesExchangeRateInverse will be lower, causing the return value from _valueOfUnderlying of tokens withdrawal to be higher.
```solidity
    function sharesExchangeRateInverse()
        public
        view
        virtual
        returns (uint256 rate_)
    {
        return

            (EXCHANGE_RATE_EXPANSION_FACTOR * EXCHANGE_RATE_EXPANSION_FACTOR) /
            sharesExchangeRate(); // @audit rate_ returned is higher causing this function return value also to be LOWER
    }
    function sharesExchangeRate() public view virtual returns (uint256 rate_) {
         -- SNIP --
        rate_ =
            (poolTotalEstimatedValue  *
                EXCHANGE_RATE_EXPANSION_FACTOR) /
            poolSharesToken.totalSupply();  //@audit supply decreased, means rate_ will be higher
            }
```
### Impact
User will have higher amount of token asset withdrawn.

### Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L403C1-L408C11
Run forge test --match-test test_burn -vv, if you want to check the error, change assertNotEq to assertEq
The test shows that a user withdraws half of 1e18 twice, and will noticed that 1e18 of tokens asset is returned.
```solidity
    function test_burn()
        public
    {
        principalToken.transfer(address(lenderCommitmentGroupSmart), 1e18);
        collateralToken.transfer(address(lenderCommitmentGroupSmart), 1e18);

        // @audit Removed to use Original ExchangeRate
        // lenderCommitmentGroupSmart.set_mockSharesExchangeRate(1e36 * 2);

        initialize_group_contract();

        //lenderCommitmentGroupSmart.set_totalPrincipalTokensCommitted(1000000);
        //lenderCommitmentGroupSmart.set_totalInterestCollected(2000000);

        vm.prank(address(lender));
        principalToken.approve(address(lenderCommitmentGroupSmart), 1e18);

        vm.prank(address(lender));
        //5e17
        uint256 sharesAmount_ = lenderCommitmentGroupSmart
            .addPrincipalToCommitmentGroup(1e18, address(borrower));

        uint256 expectedSharesAmount = 1e18;

        assertEq(
            sharesAmount_,
            expectedSharesAmount,
            "Received an unexpected amount of shares"
        );
        vm.prank(address(borrower));
        uint expectedAssetAmount = 5e17;
        uint256 assetsAmount = lenderCommitmentGroupSmart.burnSharesToWithdrawEarnings(expectedAssetAmount,address(borrower));
                assertNotEq( // Returned not Equal
            assetsAmount,
            expectedAssetAmount,
            "Received an unexpected amount of shares"
        );
        vm.prank(address(borrower));
        uint expectedAssetAmount2 = 5e17;
        uint256 assetsAmount2 = lenderCommitmentGroupSmart.burnSharesToWithdrawEarnings(expectedAssetAmount,address(borrower));
                assertNotEq(// Returned not Equal
            assetsAmount2,
            expectedAssetAmount2,
            "Received an unexpected amount of shares2"
        );
    }
```
### Tool used
Manual Review

### Recommendation
```diff
    function burnSharesToWithdrawEarnings(
        uint256 _amountPoolSharesTokens,
        address _recipient
    ) external returns (uint256) {
      
        //@audit-issue
--        poolSharesToken.burn(msg.sender, _amountPoolSharesTokens);

        uint256 principalTokenValueToWithdraw = _valueOfUnderlying(
            _amountPoolSharesTokens,
            sharesExchangeRateInverse()
        );
++        poolSharesToken.burn(msg.sender, _amountPoolSharesTokens);
```