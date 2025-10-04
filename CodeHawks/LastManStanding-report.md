# [CodeHawks Last-Man-Standing Report](https://codehawks.cyfrin.io/c/2025-07-last-man-standing)

## Findings by osok17
| Severity | Title | 
|:--:|:---|
| [H-01](#h-01-missing-grace-period-validation-in-claimthrone-enables-invalid-claims-post-expiration) | Missing Grace Period Validation in `claimThrone()` Enables Invalid Claims Post-Expiration | 
| [H-02](#h-02-overpayment-to-claimthrone-not-refunded-resulting-in-unaccounted-eth-accrual) | Overpayment to `claimThrone()` Not Refunded, Resulting in Unaccounted ETH Accrual | 
| [M-01](#m-01-incorrect-access-check-in-claimthrone-blocks-all-new-participants) | Incorrect Access Check in `claimThrone()` Blocks All New Participants | 
| [M-02](#m-02-missing-previous-king-payout-logic-in-claimthrone) | Missing Previous King Payout Logic in `claimThrone()` | 
| [L-01](#l-01-graceperiod-reset-unexpectedly-after-game-reset-causing-inconsistency-between-updated-and-initial-values) | GracePeriod reset unexpectedly after game reset, causing inconsistency between updated and initial values | 
| [L-02](#l-02-gameended-event-emits-incorrect-prizeamount-always-zero) | GameEnded event emits incorrect prizeAmount (always zero) |
| [L-03](#l-03-insufficient-increment-in-claimfee-for-very-small-initial-values-causing-fee-stagnation) | Insufficient Increment in claimFee for Very Small Initial Values Causing Fee Stagnation |


# [H-01] Missing Grace Period Validation in `claimThrone()` Enables Invalid Claims Post-Expiration

## Root + Impact
The `claimThrone()` function lacks a check to ensure the grace period has not expired, allowing users to continue claiming the throne after the game should have ended.

According to the game’s rules, once the grace period has expired without a new king, the last king should be declared the winner. However, `claimThrone()` does not verify whether the grace period has elapsed, allowing players to continue claiming the throne even after the deadline, leading to inconsistent game state and broken reward logic.

```solidity
function claimThrone() external payable {
    // Missing check for gracePeriod expiration
    currentKing = msg.sender;
```

## Risk
- Likelihood:
Any player can claim the throne after the grace period as long as no one calls `declareWinner()` to end the game.

- Impact:
1. The game cannot end properly, preventing the rightful winner from claiming the pot and degrading the user experience.
2. Potentially allows continuous throne claims, making the game unplayable or unfair.

## Proof of Concept
The test shows that after the grace period ends, a new player can still claim the throne and become king, which should not be allowed according to the game rules.
```solidity
function testCanClaimAfterGracePeriodButBeforeDeclareWinner() public {
        vm.startPrank(player1);
        game.claimThrone{value: INITIAL_CLAIM_FEE}();
        vm.stopPrank();
​
        vm.warp(block.timestamp + GRACE_PERIOD + 1);
​
        uint256 newClaimFee = game.getCurrentClaimFee();
        console2.log("new claim fee:", newClaimFee);
​
        vm.startPrank(maliciousActor);
        game.claimThrone{value: newClaimFee}();
        assertEq(game.currentKing(), maliciousActor, "Malicious actor unexpectedly became king after grace period"); 
}
```

## Recommendation
Add a grace period validation check at the beginning of `claimThrone()`:
```diff
+   require(block.timestamp <= lastClaimTime + gracePeriod, "Grace period expired. Declare winner first.");
```

# [H-02] Overpayment to `claimThrone()` Not Refunded, Resulting in Unaccounted ETH Accrual

## Root + Impact

Players become the King by sending ETH equal to or greater than the current `claimFee`. However, the contract does not handle excess ETH. Any amount sent above the `claimFee` is silently absorbed and added to the pot or platform fee. This contradicts reasonable user expectations and opens a vector for malicious frontends to induce overpayment.

```solidity
require(msg.value >= claimFee, "Game: Insufficient ETH sent");
// No refund of msg.value - claimFee
// Entire msg.value processed as fee and pot contribution
```

## Risk
- Likehood 
Occurs whenever a user sends more than `claimFee` (e.g., by UI mistake, manual tx input, or malicious frontend).

- Impact
1. User funds are silently absorbed without transparency.
2. Malicious UIs could exploit this to inflate pot or platform fees.
3. Reduces trust and introduces unfair game mechanics.

## Proof of Concept
This test demonstrates that if a user overpays when calling `claimThrone()`, the extra ETH is not refunded.
```solidity
function testExcessEthIsNotRefunded() public {
    vm.startPrank(player1);
    uint256 balanceBefore = player1.balance;
​
    // Overpay: send 0.5 ETH when only 0.1 ETH is required
    game.claimThrone{value: 0.5 ether}();
​
    uint256 balanceAfter = player1.balance;
    uint256 expectedSpending = 0.5 ether;
​
    // Confirm full amount was deducted, not just claimFee
    assertEq(balanceBefore - balanceAfter, expectedSpending, "Excess ETH was not refunded");
    vm.stopPrank();
}
```

## Recommended Mitigation
Enforce exact payment or explicitly refund the difference between `msg.value` and `claimFee`:
```solidity
//Option 1: Require exact ETH
require(msg.value == claimFee, "Game: Must send exact claim fee.");
​
//Option 2: Accept overpayment but refund
require(msg.value >= claimFee, "Game: Insufficient ETH sent.");
uint256 excess = msg.value - claimFee;
if (excess > 0) {
    payable(msg.sender).transfer(excess);
}
```

# [M-01] Incorrect Access Check in `claimThrone()` Blocks All New Participants

## Root + Impact

Any user (except the current king) should be able to call `claimThrone` by sending enough ETH to become the new king.

The intention is to prevent the current king from re-claiming the throne, not to restrict all others. However, the condition is inverted, allowing only the current king to call `claimThrone()`, and preventing all new participants.

## Risk
- Likehood 
This occurs immediately after deployment, as no one except the initial king can participate.

- Impact
1. The main game mechanic (`claimThrone`) becomes inaccessible to all users.
2. The game cannot be started or played as intended.

## Proof of Concept
This test shows that calling `claimThrone` reverts for any player who is not the current king, blocking all claims:
```solidity
function testNooneCanClaimThrone() public {
        vm.startPrank(player1);
        vm.expectRevert("Game: You are already the king. No need to re-claim.");
        game.claimThrone{value: INITIAL_CLAIM_FEE}();
}
```

## Recommended Mitigation
Update the require condition to correctly reject calls from the current king, allowing all others to participate:
```diff
-   require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");
+   require(msg.sender != currentKing, "Game: You are already the king. No need to re-claim.");
```

# [M-02] Missing Previous King Payout Logic in `claimThrone()`
## Root + Impact
Although the contract's documentation suggests that a portion of the new claim fee should be paid to the dethroned king, the logic to perform this payout is completely absent. Additionally, variables like previousKingPayout are defined but unused, reinforcing the appearance of incomplete implementation.

The `claimThrone()` function documentation states that a portion of the new claim fee should be paid to the previous king. However, this logic is entirely missing from the implementation.
The variable previousKingPayout is initialized to zero and never updated. As a result, no funds are transferred to the previous king, despite the comment suggesting otherwise.
This breaks the economic incentive model and may mislead users expecting a partial rebate when dethroned.

```solidity
function claimThrone() external payable gameNotEnded nonReentrant {
        require(msg.value >= claimFee, "Game: Insufficient ETH sent to claim the throne.");
        require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");
​
        uint256 sentAmount = msg.value;
@>      uint256 previousKingPayout = 0;
        uint256 currentPlatformFee = 0;
        uint256 amountToPot = 0;
```

## Risk
- Likehood 
The issue occurs during normal contract usage. Any player claiming the throne will trigger it, making exploitation trivial and highly likely.

- Impact
1. Misleading documentation / broken incentive mechanism.
2. Players may expect to receive compensation after being dethroned but will not.

## Proof of Concept
```solidity
function testNoPreviousKingPayout() public {
        uint256 player1BalanceBeforeClaim = address(player1).balance;
​
        vm.startPrank(player1);
        game.claimThrone{value: INITIAL_CLAIM_FEE}();
        vm.stopPrank();
        uint256 player1BalanceAfterClaim = address(player1).balance;
​
        console2.log("player1 balance before claim:", player1BalanceBeforeClaim);
        console2.log("player1 balance after claim:", player1BalanceAfterClaim);
​
        vm.startPrank(player2);
        game.claimThrone{value: 1 ether}();
        vm.stopPrank();
        
        uint256 player1BalanceAfterSecondClaim = address(player1).balance;
        console2.log("player1 balance after second claim:", player1BalanceAfterSecondClaim);
        // player1 balance before claim: 10000000000000000000 10
        // player1 balance after claim: 9900000000000000000   9.9
        // player1 balance after second claim: 9900000000000000000 9.9
    }
```

## Recommended Mitigation
Implement the payout logic to the previous king as described, e.g.:
Or alternatively, update the documentation and remove unused variables (e.g. previousKingPayout) if this behavior is not intended.
```solidity
if (currentKing != address(0)) {
    uint256 payout = (sentAmount * previousKingPercentage) / 100;
    (bool success, ) = payable(currentKing).call{value: payout}("");
    require(success, "Game: Failed to pay previous king");
    previousKingPayout = payout;
}
```

# [L-01] GracePeriod reset unexpectedly after game reset, causing inconsistency between updated and initial values
## Root + Impact
The `gracePeriod` parameter controls how long the current King can hold their throne before another player can claim it. It can be updated by the owner via `updateGracePeriod()` and is expected to persist for the duration of the game round.
While `updateGracePeriod()` correctly updates the current `gracePeriod`, calling `resetGame()` resets the gracePeriod silently back to the original initial value set at contract deployment (initialGracePeriod). This causes a mismatch between the player's expectations (based on the updated gracePeriod) and the actual game behavior after reset.

```solidity
function updateGracePeriod(uint256 _newGracePeriod) external onlyOwner {
    require(_newGracePeriod > 0, "Game: New grace period must be greater than zero.");
@>  gracePeriod = _newGracePeriod;
    emit GracePeriodUpdated(_newGracePeriod);
}
function resetGame() external onlyOwner gameEndedOnly {
    ...
@>  gracePeriod = initialGracePeriod;
    ...
}
```

## Risk
- Likehood 
Occurs whenever the owner updates the grace period during an active round, then ends the round and resets the game.

- Impact
1. Players relying on the updated grace period might be misled, affecting their strategies.
2. Gameplay fairness and predictability are compromised.
3. Could result in users missing winning conditions or unexpected game resets.

## Proof of Concept
After updating the grace period, it works as expected during the round. But when the game is reset, the grace period silently reverts to the original value. This mismatch can confuse users and affect gameplay.
```solidity
function testGracePeriodResetsUnexpectedly() public {
        // Update the grace period to a new value (2 days)
        vm.startPrank(deployer);
        uint256 newGracePeriod = 2 days;
        game.updateGracePeriod(newGracePeriod);
        vm.stopPrank();
​
        // Verify the grace period was updated correctly
        assertEq(game.getCurrentGracePeriod(), newGracePeriod);
​
        // Player1 claims the throne, and ends the current game round
        vm.startPrank(player1);
        game.claimThrone{value: INITIAL_CLAIM_FEE}();
        vm.warp(block.timestamp + newGracePeriod + 1);
        game.declareWinner();
​
        // Owner resets the game for a new round. Verify that the grace period has been reset to the initial default value
        vm.startPrank(deployer);
        game.resetGame();
        assertEq(game.getCurrentGracePeriod(), GRACE_PERIOD, "back to the old period");
    }
```

## Recommended Mitigation
Synchronize updates to initialGracePeriod inside updateGracePeriod() so that resetting the game preserves the updated value:
```diff
function updateGracePeriod(uint256 _newGracePeriod) external onlyOwner {
    require(_newGracePeriod > 0, "Game: New grace period must be greater than zero.");
-   gracePeriod = _newGracePeriod;
+   initialGracePeriod = _newGracePeriod; // Keep initial value in sync
    emit GracePeriodUpdated(_newGracePeriod);
```

# [L-02] GameEnded event emits incorrect prizeAmount (always zero)
## Root + Impact
The `GameEnded` event is intended to log the final prize awarded to the winner. However, in `declareWinner()`, the `pot` is set to zero before emitting the event. As a result, the `prizeAmount` in the event is always zero, even though the winner receives the correct amount via `pendingWinnings`.

```solidity
pendingWinnings[currentKing] += pot;
@> pot = 0;
@> emit GameEnded(currentKing, pot, block.timestamp, gameRound); // Emits 0
```

## Risk
- Likehood
This occurs every time `declareWinner()` is called after the grace period.

- Impact
1. All GameEnded logs will show a prize amount of zero.
2. Off-chain systems (e.g. UIs, indexers, analytics) relying on events will display incorrect data.
3. Reduces trust and transparency for users.

## Proof of Concept
In a typical game flow, a user claims the throne with ETH, waits for the grace period to expire, and `declareWinner()` is called. Even though the user receives the correct amount internally, the emitted event always reports the prize as zero due to resetting the pot before emitting. This behavior can be easily reproduced in any test or real scenario.

## Recommended Mitigation
Store `pot` in a local variable before resetting it, and emit that value in the event:

```solidity
uint256 prizeAmount = pot;
pendingWinnings[currentKing] += prizeAmount;
pot = 0;
emit GameEnded(currentKing, prizeAmount, block.timestamp, gameRound);
```

# [L-03] Insufficient Increment in claimFee for Very Small Initial Values Causing Fee Stagnation
## Root + Impact

The `claimFee` is designed to increase by a fixed percentage (`feeIncreasePercentage`) after each successful throne claim to incentivize progressively higher stakes.

However, when the initial `claimFee` is set to a very small value (e.g., below `100 / feeIncreasePercentage` wei), the calculated fee increment (`claimFee * feeIncreasePercentage`) / 100 evaluates to zero due to integer division rounding.

As a result, the `claimFee` remains constant across claims, breaking the intended progressive increase mechanic.

```solidity
// Increment calculation in claimThrone()
@》claimFee = claimFee + (claimFee * feeIncreasePercentage) / 100; 
```

## Risk
- Likehood
This issue only manifests if the initial claimFee is configured with a very small value, which is unlikely under typical deployment parameters.

- Impact
The stagnation of the claimFee breaks the game's economic design by allowing players to claim the throne repeatedly at a fixed minimal cost, potentially impacting user incentives and gameplay fairness.

## Proof of Concept
Deploy the contract with an initial `claimFee` of 1 wei and a `feeIncreasePercentage` of 10.

After a player successfully calls `claimThrone()` by sending 1 wei, observe that the `claimFee` remains 1 wei due to rounding of the increment to zero.

```solidity
function testClaimFeeDoesNotIncreaseDueToRounding() public {
        vm.startPrank(player1);
​
        uint256 feeBefore = game.claimFee();
        console2.log("",feeBefore);
​
        // Player1 claims throne by sending 1 wei (minimum claimFee)
        game.claimThrone{value: 1}();
​
        uint256 feeAfter = game.claimFee();
        console2.log("",feeAfter);
​
        // Because (1 * 10)/100 = 0 (integer division), feeAfter == feeBefore expected
        assertEq(feeAfter, feeBefore, "claimFee should not increase due to rounding");
​
        vm.stopPrank();
    }
```

## Recommended Mitigation
Implement a minimum increment logic to guarantee the `claimFee` increases by at least 1 wei per claim:
Alternatively, enforce a reasonable lower bound on the initial `claimFee` during contract deployment to prevent values that would cause zero increments.