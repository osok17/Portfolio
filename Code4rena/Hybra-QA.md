# [Hybra QA Report](https://code4rena.com/reports/2025-10-hybra-finance#low-risk-and-non-critical-issues)

| ID | Title |
|:--:|:---|
| [L-1](#l-1-lack-of-eip-721-safety-checks-in-_mint-can-lead-to-permanent-loss-of-locked-principal-assets) | Lack of EIP-721 safety checks in `_mint()` can lead to permanent loss of locked principal assets |
| [L-2](#l-2-increase_unlock_time-resets-lock-based-on-current-timestamp-instead-of-extending-it) | `increase_unlock_time()` resets lock based on current timestamp instead of extending it |
| [L-3](#l-3-core-functions-assume-fee-free-tokens-causing-inaccurate-accounting-and-potential-reverts) | Core functions assume fee-free tokens, causing inaccurate accounting and potential reverts |
| [L-4](#l-4-core-token-invariant-is-broken-by-transfers-to-address0-leading-to-inaccurate-protocol-calculations) | Core Token Invariant is Broken by Transfers to `address(0)`, Leading to Inaccurate Protocol Calculations |
| [L-5](#l-5-multisplit-loses-locked-tokens-due-to-integer-division-rounding) | `multiSplit()` Loses Locked Tokens Due to Integer Division Rounding |
| [L-6](#l-6-redundant-whitelist-logic-in-_beforetokentransfer-creates-unnecessary-complexity) | Redundant whitelist logic in `_beforeTokenTransfer` creates unnecessary complexity |
| [L-7](#l-7-missing-explicit-allowance-check-may-cause-unclear-revert-reason) | Missing explicit allowance check may cause unclear revert reason |
| [L-8](#l-8-incorrect-event-emitter-address-in-_getreward) | Incorrect event emitter address in `_getReward()` |
| [L-9](#l-9-invalid-zero-address-check-allows-setting-internal_bribe-to-zero-address) | Invalid zero address check allows setting `internal_bribe` to zero address |

---

# [L-1] Lack of EIP-721 safety checks in `_mint()` can lead to permanent loss of locked principal assets
https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/VotingEscrow.sol#L509

## Finding description and impact

The `_mint()` function does not perform the standard ERC-721 `onERC721Received` check. This omission has serious implications in the context of a `VotingEscrow` contract, where each minted NFT represents a user’s time-locked principal assets rather than a mere collectible.

If a user — either by mistake or through an unreliable frontend — mints the NFT to a smart contract that does not implement `onERC721Received`, the NFT will become permanently locked.
As a result, the user would not only lose access to their voting power during the lock period, but more critically, they would be unable to withdraw the underlying assets once the lock expires. This effectively leads to a permanent and unrecoverable loss of funds for the affected user.

## Recommended mitigation

Use `_safeMint()`-style logic to ensure `_to` is a valid ERC721 receiver, or restrict `_to` to EOAs only.

---

# [L-2] `increase_unlock_time()` resets lock based on current timestamp instead of extending it
https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/VotingEscrow.sol#L900

## Finding description and impact

The function calculates the new unlock time relative to the current timestamp rather than extending the existing lock. This means that calling `increase_unlock_time()` effectively resets the lock duration instead of extending it.
```solidity
function increase_unlock_time(uint _tokenId, uint _lock_duration) external nonreentrant {
    ...
    uint unlock_time = (block.timestamp + _lock_duration) / WEEK * WEEK;
```

Users may expect the lock to be extended on top of the existing duration, but it instead resets from the current timestamp. While this behavior may be intentional (to simplify lock management or limit perpetual extensions), it could cause user confusion, as the function name suggests additive behavior.

## Recommended mitigation

If the intended behavior is to reset the lock, consider: Renaming the function to `reset_unlock_time()` or Updating documentation and UI prompts to clarify its behavior.

Otherwise, if the intention is to extend, modify the calculation to:
```diff
-   uint unlock_time = (block.timestamp + _lock_duration) / WEEK * WEEK;
+   uint unlock_time = (_locked.end + _lock_duration) / WEEK * WEEK;
```

---

# [L-3] Core functions assume fee-free tokens, causing inaccurate accounting and potential reverts
https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/CLGauge/GaugeCL.sol#L304
https://github.com/code-423n4/2025-10-hybra-finance/blob/74c2b93e8a4796c1d41e1fbaa07e40b426944c44/ve33/contracts/swapper/HybrSwapper.sol#L47

## Finding description and impact

Several core functions across the protocol assume that token transfers do not charge any transfer fees. This assumption can lead to incorrect behavior if a fee-on-transfer or non-standard ERC20 token is ever used.

- In `GaugeCL._claimFees()`. The function forwards `claimed0` and `claimed1` directly to the `internal_bribe` contract, assuming they represent the full claimable fee amounts:
```solidity
            uint256 _fees0 = claimed0;
            uint256 _fees1 = claimed1;

            if (_fees0  > 0) {
                IERC20(_token0).safeApprove(internal_bribe, 0);
                IERC20(_token0).safeApprove(internal_bribe, _fees0);
                IBribe(internal_bribe).notifyRewardAmount(_token0, _fees0);
            } 
            if (_fees1  > 0) {
                IERC20(_token1).safeApprove(internal_bribe, 0);
                IERC20(_token1).safeApprove(internal_bribe, _fees1);
                IBribe(internal_bribe).notifyRewardAmount(_token1, _fees1);
            } 
```
This assumption holds for standard ERC20 tokens (like USDC or WETH) but breaks for fee-on-transfer tokens.
If such tokens are ever added to liquidity pools, `internal_bribe` will receive fewer tokens than expected, leading to desynchronization between recorded and actual rewards.

- In `HybrSwapper.swapToHYBR()`. The function also assumes no transfer fees when transferring tokens from the caller:
```solidity
IERC20(params.tokenIn).safeTransferFrom(msg.sender, address(this), params.amountIn);
IERC20(params.tokenIn).safeApprove(params.aggregator, params.amountIn);
```
If `params.tokenIn` is a fee-on-transfer token, the contract will receive fewer tokens than `amountIn`, yet still approve the aggregator for the full `amountIn`.
This can cause unexpected reverts or incorrect swap outcomes due to mismatched balances or slippage checks.

## Recommended mitigation

- To support fee-on-transfer tokens safely, compare pre- and post-transfer balances to determine the actual received amount before proceeding. 
- Use the actual received amount in further calculations and approvals.

---

# [L-4] Core Token Invariant is Broken by Transfers to `address(0)`, Leading to Inaccurate Protocol Calculations
https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/HYBR.sol#L55
https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/RewardHYBR.sol#L113

## Finding description and impact

A fundamental accounting invariant of the HYBR token can be permanently violated. The root cause is the internal `_transfer()` function in `HYBR.sol`, which lacks a critical check to prevent transfers to the zero address (`address(0)`). While tokens sent to this address are rendered irrecoverable, the function does not decrease the `totalSupply` state variable.

This underlying flaw is exposed and can be triggered through two distinct, public-facing functions in the protocol:

1. Direct Transfer via `HYBR.transfer()` – Any user or contract can call `transfer(address _to, uint _value)` and pass `address(0)` as `_to`.  
2. Redemption via `RewardHYBR.redeemFor()` – A user can call `redeemFor(..., address recipient)` and provide `address(0)` as the `recipient`. The contract does not validate the `recipient` address and proceeds to call the flawed transfer logic in HYBR on the user’s behalf.

Triggering either path leads to the same result: the reported `totalSupply()` of the HYBR token becomes higher than the actual sum of all attainable tokens, breaking the invariant `totalSupply() == sum(balanceOf(all addresses))`.

### Impact

A fundamental accounting invariant of the HYBR token can be permanently violated. The root cause is the internal `_transfer()` function in `HYBR.sol`, which lacks a critical check to prevent transfers to the zero address (`address(0)`). While tokens sent to this address are rendered irrecoverable, the function does not decrease the totalSupply state variable.

1. Inaccurate Inflation Calculation: The `RewardHYBR.circulating_emission()` function calculates a portion of the weekly emissions based on the formula `(_hybr.totalSupply() * TAIL_EMISSION) / MAX_BPS`. Because this calculation uses the incorrect `_hybr.totalSupply()` as a direct input, it will consistently compute a higher emission value than is correct according to the economic model. This results in a persistent rate of excess inflation.
2. Incorrect Reward Distribution to Stakers: The `RewardHYBR.calculate_rebase()` function determines the portion of emissions allocated to `veHYBR` stakers. It calculates a lockedShare using `_hybrTotal` (which is `_hybr.totalSupply()`) as the denominator. An inflated denominator systematically suppresses the value of the lockedShare ratio. This leads to a direct and quantifiable under-allocation of rewards to `veHYBR` holders, as they receive a smaller portion of the weekly mint than they are entitled to based on the true proportion of locked tokens.

In summary, this invariant break introduces a persistent accounting mismatch that propagates into key emission and reward logic, leading to measurable discrepancies in protocol-level calculations.

## Recommended mitigation steps

1. The `_transfer()` function in HYBR.sol must prohibit transfers to the zero address, which is standard practice for ERC20 tokens.
```diff
function _transfer(address _from, address _to, uint _value) internal returns (bool) {
+   require(_to != address(0), "HYBR: transfer to the zero address is forbidden");

    balanceOf[_from] -= _value;
    ...
```
2. The `redeemFor()` function should ensure the recipient is a valid address.
```diff
function redeemFor(uint256 amount, uint8 redeemType, address recipient) external nonReentrant whenNotPaused {
+   require(recipient != address(0), "RewardHYBR: recipient cannot be the zero address");
    // ...
```

## POC

This test demonstrates that transferring tokens to `address(0)` does not reduce the `totalSupply`. After transferring `1,000 HYBR` tokens to the zero address, the `totalsupply` remains unchanged even though the sender’s balance decreases and the zero address balance increases.

Copy the following test into `ve33/test/C4PoC.t.sol`, then run the command:`forge test --mt test_transferToZero_breaksInvariant -vv`
```solidity
function test_transferToZero_breaksInvariant() public {
        uint256 amount = 1_000 * 1e18;

        uint256 initialSupply = hybr.totalSupply();
        uint256 initialZero = hybr.balanceOf(address(0));
        uint256 initialSender = hybr.balanceOf(deployer);

        hybr.transfer(address(0), amount);

        uint256 finalSupply = hybr.totalSupply();
        uint256 finalZero = hybr.balanceOf(address(0));
        uint256 finalSender = hybr.balanceOf(deployer);

        console.log("Transfer amount:", amount);
        console.log("Initial totalSupply:", initialSupply);
        console.log("Final totalSupply:  ", finalSupply);
        console.log("delta totalSupply:      ", int256(finalSupply) - int256(initialSupply));
        console.log("balanceOf[0x0]:   ", int256(finalZero) - int256(initialZero));
        console.log("sender balance:   ", int256(finalSender) - int256(initialSender));

        assertEq(finalSupply, initialSupply, "BUG: totalSupply should decrease");
        assertEq(finalZero, initialZero + amount, "0x0 balance mismatch");
        assertEq(finalSender, initialSender - amount, "sender balance mismatch");
    }
```

---

# [L-5] `multiSplit()` Loses Locked Tokens Due to Integer Division Rounding
https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/VotingEscrow.sol#L1184

## Finding description and impact

The `multiSplit()` function splits a single veNFT into multiple smaller veNFTs based on user-specified weights. However, because Solidity integer division truncates toward zero, the total sum of the resulting split amounts can be less than the original locked amount, causing permanent token loss (“dust”).
```solidity
amount: int128(int256(uint256(int256(originalLocked.amount)) * amounts[i] / totalWeight)),
```

A few wei may be permanently lost when splitting, due to rounding.This is a minor accounting inaccuracy and not exploitable, but technically results in user value loss.

## Recommended mitigation

Consider assigning the remainder to the last split or rounding up one sub-lock to ensure that the total equals the original amount.

## POC
Copy the following test into `ve33/test/C4PoC.t.sol`, then run the command:`forge test --mt test_PoC_MultiSplitLosesDust`
```solidity
function test_PoC_MultiSplitLosesDust() public {
    // 1. Setup: Lock an amount guaranteed to cause rounding errors when divided by 3.
    uint256 originalAmount = 100 ether;
    hybr.approve(address(votingEscrow), originalAmount);
    uint256 originalTokenId = votingEscrow.create_lock(originalAmount, block.timestamp + 4 weeks);

    votingEscrow.toggleSplit(address(this), true); // We authorize ourselves (address(this)) to pass the `splitAllowed` modifier.

    // 2. Action: Split the NFT into 3 equal parts.
    uint[] memory weights = new uint[](3);
    weights[0] = 1;
    weights[1] = 1;
    weights[2] = 1;
        
    uint256[] memory newTokenIds = votingEscrow.multiSplit(originalTokenId, weights);

     // 3. Assertion: Verify the sum of the new amounts is less than the original.
    uint256 sumOfNewAmounts = 0;
    for (uint i = 0; i < newTokenIds.length; i++) {
        (int128 newAmount, ,) = votingEscrow.locked(newTokenIds[i]);
        sumOfNewAmounts += uint128(newAmount);
     }
    // The core assertion
    assertTrue(sumOfNewAmounts < originalAmount);
}
```

---

# [L-6] Redundant whitelist logic in `_beforeTokenTransfer` creates unnecessary complexity
https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/RewardHYBR.sol#L260

## Finding description and impact

The `RewardHYBR` token is designed to be non-transferable, as both `transfer()` and `transferFrom()` revert. Therefore, tokens can only move through `minting (from == address(0)) or burning (to == address(0))`. However, the `_beforeTokenTransfer()` hook implements a redundant whitelist system (`exempt`, `exemptTo`, and `isGauge`) that is never actually used. This is because the helper function `_isExempted()` always returns `true` for mint or burn operations — the only cases where `_beforeTokenTransfer()` is triggered.
```solidity
if (_isExempted(from, to)) {
    allowed = 1;
} else if (gaugeManager != address(0) && IGaugeManager(gaugeManager).isGauge(from)) {
    exempt.add(from);
    allowed = 1;
}
if (allowed != 1) revert TransferNotAllowed();
```
As a result:
1. The whitelist mechanism has no practical effect, since user-to-user transfers are disabled and whitelisted addresses cannot transfer tokens anyway.
2. The logic adds unnecessary complexity and gas overhead while providing no functional or security benefit.

## Recommended mitigation

The `_beforeTokenTransfer()` function should be simplified to reflect the actual transfer rules ("mints and burns only") by removing the entire redundant whitelist system.
The associated state variables and management functions for `exempt` and `exemptTo` should also be removed.

---

# [L-7] Missing explicit allowance check may cause unclear revert reason
https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/HYBR.sol#L68

## Finding description and impact

The `transferFrom()` function does not explicitly check if `_value` exceeds the spender’s allowance before subtraction. When `_value > allowance[_from][msg.sender]`, the transaction reverts due to an arithmetic underflow with the generic panic code 0x12, instead of reverting with a clear and informative error message (e.g., "ERC20: insufficient allowance").

This reduces debuggability and may confuse users or integrators when transactions revert unexpectedly without a readable reason.

## Recommended mitigation

Add an explicit check for sufficient allowance before subtraction to provide a clear revert message and improve contract transparency:
```solidity
if (allowed_from != type(uint).max) {
    require(_value <= allowed_from, "ERC20: insufficient allowance");
    allowance[_from][msg.sender] = allowed_from - _value;
}
```

---

# [L-8] Incorrect event emitter address in `_getReward()`
https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/CLGauge/GaugeCL.sol#L236

## Finding description and impact

The `_getReward()` function emits the `Harvest` event using `msg.sender` instead of the actual beneficiary `account`. When called via `getReward()` (which uses the `onlyDistribution` modifier), `msg.sender` refers to the distribution contract rather than the user who receives the reward.
This leads to misleading event logs and may cause indexers or analytics systems to attribute rewards to the wrong address.

## Recommended mitigation

Update the event emission to correctly reflect the actual beneficiary:
```solidity
emit Harvest(account, rewardAmount);
```

---

# [L-9] Invalid zero address check allows setting `internal_bribe` to zero address
https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/CLGauge/GaugeCL.sol#L344
https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/GaugeV2.sol#L131

## Finding description and impact

The function `setInternalBribe()` intends to prevent setting the internal bribe contract (`internal_bribe`) to the zero address.
However, the check is incorrect:
```solidity
require(_int >= address(0), "zero");
```
This condition always evaluates to true, since all addresses are greater than or equal to `address(0)`.
As a result, the owner could accidentally set `_int` to the zero address, which may cause fee transfers to fail or lead to funds being lost.

> Note: The same issue also exists in `GaugeV2`, but it was already reported in the [Blackhole-L13](https://code4rena.com/reports/2025-05-blackhole#13-incorrect-require-statement-for-address-check-at-gaugev2sol). We include it here only for completeness and confirmation that the bug remains unfixed in the current codebase.

## Recommended mitigation

Use a proper non-zero address check:
```solidity
require(_int != address(0), "zero");
```




