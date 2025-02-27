<table>
    <tr><th></th><th></th></tr>
    <tr>
        <td><img src="../assets/mondrian-wallet-v2.png" width="250" height="250" /></td>
        <td>
            <h1>First Flight #19: Mondrian Wallet v2</h1>
            <h2>zkSync Account Abstraction</h2>
            <p>Prepared by: 0xjoaovictor</p>
            <p>Date: July 11 2024</p>
        </td>
    </tr>
</table>

# About **First Flight #19: Mondrian Wallet v2**

The Mondrian Wallet team is back! And they decided "oh wow, zkSync has native account abstraction! Let's just use that. Also, we introduced a lot of bugs, so let's just make this codebase upgradeable, so that only the owner of the wallet can introduce functionality later as they see fit. Also, the NFT gimmick was silly so, let's not do that again."

If the contracts are upgradeable, we'll just be able to upgrade them if there is a bug, so no issues right?

To *really* understand this codebase, you'll want to learn about:

* Account Abstraction
* zkSync System Contracts
* Upgradable smart contracts via UUPS

# Summary & Scope

The following contracts were in scope:
```solidity
./src/
#-- MondrianWallet2.sol
```

# Summary of Findings

| ID     | Title                        | Severity      | Fixed |
| ------ | ---------------------------- | ------------- | ----- |
| [H-01] | `MondrianWallet2::executeTransactionFromOutside` don't verify if transaction is valid | High | ✓ |
| [H-02] | `MondrianWalletV2::_authorizeUpgrade` allow anyone upgrade the contract| High | ✓ |
| [H-03] | `MondrianWalletV2` can't receive ETH transfer | High | ✓ |

# Detailed Findings

## [H-01] `MondrianWallet2::executeTransactionFromOutside` don't verify if transaction is valid

#### Summary

When `MondrianWallet2::executeTransactionFromOutside` is called, the function don't verify if transaction is valid.

#### Vulnerability Details

We can see below that `MondrianWallet2::executeTransactionFromOutside` calls `_validateTransaction` but don't verify if the return is valid:

```solidity
function executeTransactionFromOutside(Transaction memory _transaction) external payable {
        _validateTransaction(_transaction);
        _executeTransaction(_transaction);
}
```

### Inpact

Without this verification anyone can execute the transaction becasue the signature is not verified and don't have the verification to check the required balance.

#### Tools Used

Solidity and Foundry

#### Proof of Concept

Add the following PoC to the `test/ModrianWallet2Test.t.sol`:

```solidity
    function testZkExecuteOnlyValidTransactionsOnExecuteTransactionFromOutside() public {
        // Arrange
        address anonymousUser = makeAddr("anonymousUser");
        address dest = address(usdc);
        uint256 value = 0;
        bytes memory functionData = abi.encodeWithSelector(ERC20Mock.mint.selector, address(mondrianWallet), AMOUNT);

        // Create a transaction with wrong signature
        Transaction memory transaction =
            _createUnsignedTransaction(anonymousUser, 113, dest, value, functionData);

        bytes32 unsignedTransactionHash = MemoryTransactionHelper.encodeHash(transaction);
        uint8 v;
        bytes32 r;
        bytes32 s;
        uint256 fakeKey = 0xac293483920;
        (v, r, s) = vm.sign(fakeKey, unsignedTransactionHash);
        transaction.signature = abi.encodePacked(r, s, v);

        // Act
        vm.prank(anonymousUser);

        // Expect revert: MondrianWallet2__InvalidSignature();
        vm.expectRevert(0x59f73058);

        mondrianWallet.executeTransactionFromOutside(transaction);
    }
```
And run: `forge test --zksync --system-mode=true --match-test testZkExecuteOnlyValidTransactionsOnExecuteTransactionFromOutside`

### Recommendations

Verify the return of `_validateTransaction`:

```diff
    function executeTransactionFromOutside(Transaction memory _transaction) external payable {
-          _validateTransaction(_transaction);
+         bytes4 magic = _validateTransaction(_transaction);
+         if (magic != ACCOUNT_VALIDATION_SUCCESS_MAGIC) {
+            revert MondrianWallet2__InvalidSignature();
+         }
         _executeTransaction(_transaction);
    }
```

## [H-02] `MondrianWalletV2::_authorizeUpgrade` allow anyone upgrade the contract

### Summary

The function `MondrianWalletV2::_authorizeUpgrade` don't have a verification to know who can upgrade the contract.

### Vulnerability Details

The function `MondrianWalletV2::_authorizeUpgrade` don't verify if only owner could be upgrade the contract or other actor:

```solidity
    // Needed for UUPS
    function _authorizeUpgrade(address newImplementation) internal override{}
```

### Impact

Anyone can upgrade the contract and hack the funds.

### Tools Used

* Solidity and Foundry

### Proof of Concept

Add the following PoC to `test/ModrianWallet2Test.t.sol`:

```solidity
    function testZkOnlyAdminCanUpgradeContract() public {
        // Arrange
        address anonymousUser = makeAddr("anonymousUser");
        MondrianWallet2 newImplementation = new MondrianWallet2();

        // Act
        // User that is not the owner tries to upgrade the contract
        vm.prank(anonymousUser);
        vm.expectRevert();
        UUPSUpgradeable(address(mondrianWallet)).upgradeToAndCall(address(newImplementation), bytes(""));

        // User that is the owner tries to upgrade the contract
        vm.prank(ANVIL_DEFAULT_ACCOUNT);
        UUPSUpgradeable(address(mondrianWallet)).upgradeToAndCall(address(newImplementation), bytes(""));
    }
```
And run: `forge test --zksync --system-mode=true --match-test testZkOnlyAdminCanUpgradeContract`


### Recommendations

Add the modifier `onlyOwner`:

```diff
-  function _authorizeUpgrade(address newImplementation) internal override {}
+  function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
```

## [H-03] `MondrianWalletV2` can't receive ETH transfer

### Summary

The smart contract wallet `MondrianWallet2` can't receive ETH transfer.

### Vulnerability Details

`MondrianWallet2` don't have `receive` payable function to allow receive ETH transfer.

### Impact

Without the `receive` function not will be possible transfer ETH to smart contract wallet to pay fees, hold, transfer to others address etc.

### Tools Used

Solidity and Foundry

### Proof of Concept

Add the following PoC to `test/ModrianWallet2Test.t.sol`:

```solidity
    function testZkContractCanReceiveETH() public {
        (bool sent, bytes memory data) = address(mondrianWallet).call{value: 1 ether}("");

        assertEq(sent, true);
    }
```
Run: `forge test --zksync --system-mode=true --match-test testZkContractCanReceiveETH -vvv`

### Recommendations

The smart contract wallet `MondrianWallet2` should be have the `receive` payable function:

```diff
+   receive() external payable {}

    // Needed for UUPS
    function _authorizeUpgrade(address newImplementation) internal override {}
```