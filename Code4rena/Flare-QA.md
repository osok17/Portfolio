# [Flare QA Report](https://code4rena.com/reports/2025-08-flare-fasset#low-risk-and-non-critical-issues)

| ID | Title |
|:--:|:---|
| [L-1](#l-1-lack-of-event-emission-for-state-changes) | Lack of Event Emission for State Changes |
| [L-2](#l-2-incorrect-strict-inequality-used-for-timelock-check) | Incorrect Strict Inequality Used for Timelock Check |
| [L-3](#l-3-missing-event-emission-on-partial-collateral-withdrawal) | Missing Event Emission on Partial Collateral Withdrawal |
| [L-4](#l-4-potential-gas-exhaustion-dos-in-freebalancenegativechallenge) | Potential Gas Exhaustion DoS in `freeBalanceNegativeChallenge()` |
| [L-5](#l-5-incorrect-use-of-equality-check-on-a-bitmask-field) | Incorrect Use of Equality Check on a Bitmask Field |
| [L-6](#l-6-silent-return-instead-of-revert-in-asset-manager-addremove-functions) | Silent Return Instead of Revert in Asset Manager Add/Remove Functions |
| [L-7](#l-7-destination-address-string-comparison-is-case-sensitive) | Destination Address String Comparison is Case-Sensitive |

# [L-1] Lack of Event Emission for State Changes
https://github.com/code-423n4/2025-08-flare/blob/b703ea27ee98e488d245083c63011cdbf43a74c4/contracts/assetManager/facets/AgentAlwaysAllowedMintersFacet.sol#L12
https://github.com/code-423n4/2025-08-flare/blob/main/contracts/assetManagerController/implementation/AssetManagerController.sol

## Finding description and impact

Critical administrative functions in `AgentAlwaysAllowedMintersFacet.sol` (`addAlwaysAllowedMinterForAgent()`, `removeAlwaysAllowedMinterForAgent()`) and `AssetManagerController.sol` (`addAssetManager()`, `removeAssetManager()`, `addEmergencyPauseSender()`, `removeEmergencyPauseSender()`) modify access control and configuration state without emitting events. This reduces observability, hinders off-chain monitoring, and weakens the on-chain audit trail. Emitting events for these functions is recommended.

## Recommended mitigation

It is strongly recommended to define and emit an event in each function that modifies the contract's state.


# [L-2] Incorrect Strict Inequality Used for Timelock Check
https://github.com/code-423n4/2025-08-flare/blob/b703ea27ee98e488d245083c63011cdbf43a74c4/contracts/assetManager/facets/AgentVaultManagementFacet.sol#L168

## Finding description and impact

The `destroyAgent()` function implements a timelock to ensure an agent can only be destroyed after a predefined announcement period. The check is implemented as `require(block.timestamp > agent.destroyAllowedAt, ...)`. This uses a strict inequality (`>`), which means the transaction will fail even if `block.timestamp` is exactly equal to `agent.destroyAllowedAt`. The agent owner is forced to wait for the next block after the timelock has expired.

This is an off-by-one error in the time domain. While it does not lead to a loss of funds, it results in a negative user experience by forcing users to wait one block longer than necessary. The standard and expected behavior for timelocks is to use inclusive inequality (`>=`), allowing the action to be performed as soon as the `allowedAt` timestamp is reached.

## Recommended mitigation

Change the strict inequality (`>`) to an inclusive inequality (`>=`) to align with standard practice and the intended logic of the timelock.

```diff
- require(block.timestamp > agent.destroyAllowedAt, DestroyNotAllowedYet());
+ require(block.timestamp >= agent.destroyAllowedAt, DestroyNotAllowedYet());
```


# [L-3] Missing Event Emission on Partial Collateral Withdrawal
https://github.com/code-423n4/2025-08-flare/blob/b703ea27ee98e488d245083c63011cdbf43a74c4/contracts/assetManager/facets/AgentCollateralFacet.sol#L89

## Finding description and impact

The contract shows inconsistent event logging during the withdrawal process:

- During the announcement phase, announceAgentPoolTokenRedemption → _announceWithdrawal emits an event, allowing off-chain systems to track the intended withdrawal amount and unlock time.
- However, during the execution phase, beforeCollateralWithdrawal allows the agent to perform partial withdrawals against the announced amount. While internal state variables withdrawal.amountWei and withdrawal.allowedAt are updated (reduced or cleared), no event is emitted to reflect these changes.

As a result, off-chain observers cannot know the exact amounts and timing of each partial withdrawal, and only see the initial announcement and the eventual clearance of the withdrawal state. This leads to incomplete off-chain visibility when withdrawals are executed in multiple steps.

## Recommended mitigation

Emit an event in beforeCollateralWithdrawal whenever a withdrawal is executed, including details such as the agent vault, token, withdrawn amount, and remaining announced amount. This will ensure accurate tracking of both announcement and execution phases of collateral withdrawals.
```diff
+   event WithdrawalExecuted(address agentVault, address token, uint256 amount, uint256 remaining);
```


# [L-4] Potential Gas Exhaustion DoS in `freeBalanceNegativeChallenge()`
https://github.com/code-423n4/2025-08-flare/blob/b703ea27ee98e488d245083c63011cdbf43a74c4/contracts/assetManager/facets/ChallengesFacet.sol#L143

## Finding description and impact
The function `freeBalanceNegativeChallenge()` iterates over all `_payments` provided by the caller.
Inside the outer loop, it contains another nested loop to check for duplicate `transactionId`, resulting in **O(n²) complexity**.

An attacker can supply a very large array of `_payments`, causing the function execution to consume excessive gas and potentially revert.
Since this function is `external` and affects the liquidation logic, it can create a **denial-of-service (DoS) vector**, where valid challenges may fail because the transaction runs out of gas.
This could allow a malicious agent to avoid liquidation even when conditions are met, reducing the security of the protocol.

- **Attack Scenario**:
  1. A malicious agent puts itself in a state that would normally be challenged via `freeBalanceNegativeChallenge` (e.g., free balance becomes negative).
  2. A legitimate challenger detects this and prepares a valid challenge transaction.
  3. The malicious agent monitors the mempool and submits a "poison transaction" with a very large `_payments` array at a higher gas price.
  4. Due to the O(n²) complexity, this poison transaction consumes excessive gas and reverts.
  5. Miners prioritize the high-gas-price poison transaction, temporarily blocking the legitimate challenge from being included in a block.

- **Impact**:
  - Legitimate challenges may fail or be delayed, allowing the malicious agent to temporarily **avoid liquidation**.
  - The attacker's own gas is wasted, but the **system's critical liquidation logic can be disrupted**.
  - In time-sensitive protocols, this delay could be exploited for additional financial gain.

## Recommended mitigation
- Replace the nested loop with a more gas-efficient structure, such as using a temporary `mapping(bytes32 => bool)` to track seen `transactionId`.
- Consider placing a **reasonable upper bound** on `_payments.length` to limit user input size.
- Gas-heavy challenge verification should be designed carefully to ensure that attackers cannot prevent legitimate challenges by bloating calldata.

# [L-5] Incorrect Use of Equality Check on a Bitmask Field
https://github.com/code-423n4/2025-08-flare/blob/b703ea27ee98e488d245083c63011cdbf43a74c4/contracts/assetManager/facets/LiquidationFacet.sol#L205

## Finding description and impact

The function `_agentResponsibilityWei()` incorrectly uses strict equality checks (==) to inspect the `_agent.collateralsUnderwater` state variable. This variable is intended to function as a bitmask, where multiple state flags (e.g., `Agent.LF_VAULT`, `Agent.LF_POOL`) can be active simultaneously by combining them with a bitwise OR. The correct method to check for the presence of a flag in a bitmask is with a bitwise AND (&) operation, not an equality comparison.

While the current if/else if/else structure coincidentally handles the existing state combinations (1, 2, and 3), this implementation is extremely fragile and logically flawed. This constitutes a latent bug that will cause critical failures if the protocol is ever extended with new state flags. For example, if a new state `LF_SYSTEMIC = 4` were added, a combined state of `LF_VAULT | LF_SYSTEMIC` (value 5) would be misclassified by the current logic, leading to incorrect financial liability calculations for the agent. This undermines a core safety mechanism and makes future upgrades unsafe.

## Recommended mitigation

Refactor the function to use bitwise AND (&) operations to check for flags within the collateralsUnderwater bitmask. This will ensure the logic is robust, correctly identifies all possible state combinations, and is safely extensible for future protocol upgrades.

# [L-6] Silent Return Instead of Revert in Asset Manager Add/Remove Functions
https://github.com/code-423n4/2025-08-flare/blob/b703ea27ee98e488d245083c63011cdbf43a74c4/contracts/assetManagerController/implementation/AssetManagerController.sol#L79

## Finding description and impact

The administrative functions `addAssetManager()` and `removeAssetManager()` are designed to manage the list of active asset managers. However, they contain a significant design flaw in their error-handling logic. Instead of reverting when a precondition is not met (e.g., attempting to add an already-existing manager or remove a non-existent one), the functions silently return. This results in a transaction that is successfully mined and reported as "success" to the caller, despite performing no state change.

1. `addAssetManager()`: Governance may mistakenly assume the operation succeeded, while the intended manager was not added, potentially causing downstream issues.

2. `removeAssetManager()`: In emergencies, a minor address typo would still result in a “successful” transaction, giving a false sense of security while the malicious manager remains active, posing a direct security risk.

In both cases, the lack of a revert provides misleading feedback, undermining the reliability and safety of protocol administration.

## Recommended mitigation

Replace the silent return with explicit revert statements that provide informative error messages:

```diff
    function addAssetManager(IIAssetManager _assetManager)
        external
        onlyGovernance
    {
-       if (assetManagerIndex[address(_assetManager)] != 0) return;
+       require(assetManagerIndex[address(_assetManager)] == 0, "AssetManager: already exists");

    function removeAssetManager(IIAssetManager _assetManager)
        external
        onlyGovernance
    {
        uint256 position = assetManagerIndex[address(_assetManager)];
-       if (position == 0) return;
+       require(position != 0, "AssetManager: not found");
```

# [L-7] Destination Address String Comparison is Case-Sensitive

## Finding description and impact

The functions `addAllowedDestinationAddresses()`, `removeAllowedDestinationAddresses()`, `requestTransferFromCoreVault()`, and `cancelTransferRequestFromCoreVault()` use string representations of destination addresses as identifiers. These strings are used in mappings and comparisons by computing keccak256 hashes of the strings.
The core issue is that Ethereum addresses are case-insensitive, but the current implementation treats strings as case-sensitive. This creates the following potential problems:

Two strings representing the same logical address but with different letter cases (e.g., "0xAbC..." vs "0xabc...") will produce different hashes.
This may lead to:
-    Duplicate allowed destination entries.
-    Multiple transfer requests for the same logical address.
-    Failed cancellation of transfer requests if the string case differs.

## Recommended mitigation

1. Convert _destinationAddress strings to lowercase before using them in mappings or comparisons.

2. Replace string identifiers with address type for all destination address references.

