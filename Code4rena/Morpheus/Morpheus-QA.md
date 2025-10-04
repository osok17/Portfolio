# [Morpheus QA Report](https://code4rena.com/reports/2025-08-morpheus#low-risk-and-non-critical-issues)

## Findings by osok17
| ID | Title |
|:--:|:---|
| [L-1](#l-1-lack-of-reward-pool-management-functions) | Lack of Reward Pool Management Functions |
| [L-2](#l-2-sendwsteth-lacks-amount-flexibility-forcing-all-or-nothing-transfers) | `sendWstETH` Lacks Amount Flexibility, Forcing All-or-Nothing Transfers |
| [L-3](#l-3-missing-event-emission-in-editparams-function) | Missing Event Emission in `editParams` Function |
| [L-4](#l-4-public-collectfees-function-could-disrupt-owner-operations-griefing-risk) | Public `collectFees` Function Could Disrupt Owner Operations (Griefing Risk) |
| [L-5](#l-5-fragile-array-alignment-in-manageusersinprivaterewardpool-may-lead-to-incorrect-stake-management) | Fragile Array Alignment in `manageUsersInPrivateRewardPool` May Lead to Incorrect Stake Management |


# [L-1] Lack of Reward Pool Management Functions
https://github.com/code-423n4/2025-08-morpheus/blob/a65c254e4c3133c32c05b80bf2bd6ff9eced68e2/contracts/capital-protocol/RewardPool.sol#L39

## Finding description and impact
The RewardPool contract only provides the `addRewardPool` function and does not allow modification (updateRewardPool) or deactivation of existing reward pools. Once a reward pool is added, its parameters are permanent. If an incorrect configuration is added by the owner, it cannot be corrected on-chain. This introduces an operational risk and potential unintended behavior in reward calculations, which may impact users.

## Recommended mitigation steps
To address this issue, I recommend implementing management functions that allow the owner to:

Implement a function to modify existing reward pool parameters by index.
Introduce a soft-deactivation mechanism (e.g., an isActive flag) to disable misconfigured pools without requiring a contract upgrade.

# [L-2] `sendWstETH` Lacks Amount Flexibility, Forcing All-or-Nothing Transfers
https://github.com/code-423n4/2025-08-morpheus/blob/a65c254e4c3133c32c05b80bf2bd6ff9eced68e2/contracts/capital-protocol/L1SenderV2.sol#L154

## Finding description and impact
The `sendWstETH` function transfers the contract’s entire wstETH balance from L1 to L2 via the Arbitrum Bridge. It does not support specifying a partial amount, forcing full-balance transfers. This design limits operational flexibility, complicates liquidity management. The existing parameters—`gasLimit_`, `maxFeePerGas_`, and `maxSubmissionCost_`—only control L2 execution and retryable ticket costs, not the transfer amount.

## Recommended mitigation steps
Modify the function to accept an _amountToSend parameter. Ensure _amountToSend is greater than zero and does not exceed the contract’s wstETH balance. Use _amountToSend as the transfer amount instead of the full balance.

# [L-3] Missing Event Emission in `editParams` Function
https://github.com/code-423n4/2025-08-morpheus/blob/a65c254e4c3133c32c05b80bf2bd6ff9eced68e2/contracts/capital-protocol/L2TokenReceiverV2.sol#L50

## Finding description and impact
The editParams function modifies critical swap configuration parameters without emitting an event. This reduces on-chain transparency, making it difficult for off-chain services and auditors to track parameter changes.

## Recommended mitigation steps
Emit an event whenever swap parameters are modified

# [L-4] Public `collectFees` Function Could Disrupt Owner Operations (Griefing Risk)
https://github.com/code-423n4/2025-08-morpheus/blob/a65c254e4c3133c32c05b80bf2bd6ff9eced68e2/contracts/capital-protocol/L2TokenReceiverV2.sol#L123

## Finding description and impact
The `collectFees` function, which collects accrued fees from a Uniswap V3 LP position, is declared as external without any access control. This allows any user to call it for any tokenId owned by the contract.While this does not risk theft (fees are sent to the contract itself), it can interfere with the owner’s multi-step administrative operations. For example, if the owner executes a transaction sequence that starts with collectFees followed by increaseLiquidity or swap, an attacker could front-run the collectFees call. This would alter the LP state, potentially causing the owner’s transaction to revert and wasting gas.

## Recommended mitigation steps
Add the onlyOwner modifier to collectFees to restrict access and prevent griefing attacks.

# [L-5] Fragile Array Alignment in `manageUsersInPrivateRewardPool` May Lead to Incorrect Stake Management
https://github.com/code-423n4/2025-08-morpheus/blob/a65c254e4c3133c32c05b80bf2bd6ff9eced68e2/contracts/capital-protocol/DepositPool.sol#L188

## Finding description and impact
The function `manageUsersInPrivateRewardPool` relies on four separate arrays (`users_`, `amounts_`, `claimLockEnds_`, `referrers_`) being perfectly aligned in order to update user staking data. The contract enforces only length equality, without any cross-validation of array content. Any misalignment during an administrative call can result in assigning incorrect staking amounts, lock periods, or referrers to users, leading to inaccurate reward distribution and potential protocol accounting inconsistencies.

## Recommended mitigation steps
Replace parallel arrays with a single array of structs containing all required fields. Alternatively, implement additional validation checks to mitigate misalignment risks.

