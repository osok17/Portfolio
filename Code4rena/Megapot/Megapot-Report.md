# [Megapot Report](https://code4rena.com/reports/2025-11-megapot)

| ID | Title |
|:--:|:---|
| [M-1](#m-1-emergencyrefundtickets-uses-global-referralfee-causing-inaccurate-refund-amounts) | `emergencyRefundTickets()` Uses Global `referralFee`, Causing Inaccurate Refund Amounts |

# [M-1] `emergencyRefundTickets()` Uses Global `referralFee`, Causing Inaccurate Refund Amounts
https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L592
https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L1059

## Finding description

The protocol allows the owner to update the global `referralFee` at any time via the `setReferralFee()` function. However, this value is not snapshotted in the `drawingState`, possibly by design.
```solidity
function setReferralFee(uint256 _referralFee) external onlyOwner {
        if (_referralFee > PRECISE_UNIT) revert JackpotErrors.InvalidReferralFee();
        uint256 oldReferralFee = referralFee;
        referralFee = _referralFee;
```
This becomes problematic in the `emergencyRefundTickets()` function. When calculating the refund amount for a ticket purchased with a referrer, the function reads the current global `referralFee`, instead of the `referralFee` that was in effect when the ticket was purchased.
```solidity
function emergencyRefundTickets(uint256[] memory _userTicketIds) external nonReentrant onlyEmergencyMode {
    // ...
    uint256 refundAmount = ticketInfo.referralScheme == bytes32(0) ? drawingState[ticketInfo.drawingId].ticketPrice 
                : drawingState[ticketInfo.drawingId].ticketPrice * (PRECISE_UNIT - referralFee) / PRECISE_UNIT; // Uses global referralFee
    totalRefundAmount += refundAmount;
    // ...
```
As a result, if the `referralFee` has changed between the purchase and refund, the refund amount will be inaccurate.

## Impact

- Inaccurate refund calculation: If `referralFee` changes from 15% to 30% after the ticket was purchased, the user would receive less than they originally paid. Conversely, if the fee decreases, they might receive an unintended bonus.
- Breaks immutability of rules: The refund logic effectively allows retrospective changes to refund conditions. This contradicts the protocol’s usual design pattern (e.g., for `ticketPrice` and `referralWinShare`), where parameters are snapshotted per drawing to ensure rule consistency.

## Recommended mitigation steps

- If the protocol intends to make all core parameters immutable within a drawing (including `referralFee`):
  1. Snapshot the global `referralFee` in `_setNewDrawingState()` when a new drawing starts.
  2. Make `buyTickets()` reference `drawingState[currentDrawingId].referralFee` when calculating costs.
  3. Make `emergencyRefundTickets()` reference `drawingState[ticketInfo.drawingId].referralFee` when calculating refunds.

- If flexibility in `referralFee` is desired, `emergencyRefundTickets()` must still not use the global value, since it processes past tickets.
A reasonable approach is to store the fee used at purchase time within the `TrackedTicket` struct in the `JackpotTicketNFT.sol` contract:
```diff
struct TrackedTicket {
    uint256 drawingId;
    uint256 packedTicket;
    bytes32 referralScheme;
+   uint256 referralFeeAtPurchase; // 例如
}
```

## POC
This PoC demonstrates that changing `referralFee` after ticket purchase leads to inaccurate refund amounts in `emergencyRefundTickets()`, causing users to receive a refund that does not match what they originally paid.
- Copy the test below into `test/poc/C4PoC.spec.ts`, then run `npx hardhat test test/poc/C4PoC.spec.ts`
```typescript
it("PoC: Changing referralFee cause incorrect refundAmount", async () => {
        // 1. Setup: A user buys a ticket with a referrer under the initial fee.
        const initialReferralFee = await jackpot.referralFee();
    
        await jackpotSystem.usdcMock.mint(buyerOne.address, usdc(100));
        await jackpotSystem.usdcMock.connect(buyerOne.wallet).approve(jackpot.getAddress(), ethers.MaxUint256);
    
        const ticketsToBuy: Ticket[] = [{ normals: [1n, 2n, 3n, 4n, 5n], bonusball: 1n }];
        const tx = await jackpot.connect(buyerOne.wallet).buyTickets(
            ticketsToBuy, 
            buyerOne.address, 
            [referrerOne.address], // With a referrer
            [ethers.parseEther("1")], 
            ethers.ZeroHash
        );
        const receipt = await tx.wait();
        const boughtTicketIds = receipt!.logs
            .map(log => {
                try {
                    const parsedLog = jackpot.interface.parseLog(log);
                    if (parsedLog?.name === "TicketPurchased") return parsedLog.args.userTicketId;
                } catch (e) {}
                return null;
            })
            .filter(id => id !== null);
        
        // 2. Action: Owner enables emergency mode and changes the referral fee.
        await jackpot.connect(owner.wallet).enableEmergencyMode();
    
        const manipulatedReferralFee = ethers.parseEther("0.30"); // Increase fee to 30%
        await jackpot.connect(owner.wallet).setReferralFee(manipulatedReferralFee);
    
        // 3. Verification: The user's refund is unfairly reduced.
        const aliceBalanceBefore = await jackpotSystem.usdcMock.balanceOf(buyerOne.address);
        await jackpot.connect(buyerOne.wallet).emergencyRefundTickets(boughtTicketIds);
        const aliceBalanceAfter = await jackpotSystem.usdcMock.balanceOf(buyerOne.address);
    
        const ticketPrice = jackpotSystem.deploymentParams.ticketPrice;
        
        const expectedRefund = ticketPrice * (ethers.parseEther("1") - initialReferralFee) / ethers.parseEther("1");
        const actualRefund = aliceBalanceAfter - aliceBalanceBefore;
    
        // 4. Print evidence and assert.
        expect(actualRefund).to.be.lessThan(expectedRefund);
        console.log(`Expected refund (based on original ${Number(initialReferralFee) / 1e16}% fee): ${ethers.formatUnits(expectedRefund, 6)} USDC`);
        console.log(`Actual refund (after fee increased to 30%):   ${ethers.formatUnits(actualRefund, 6)} USDC`);
    });
```
