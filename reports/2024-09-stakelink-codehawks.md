# Liquid Staking - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. No LSTs transfer on node operator withdrawals resulting in stuck funds and loss for node operators](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Remove splitter will always revert if there are some rewards left on splitter contract](#M-01)
- ## Low Risk Findings
    - ### [L-01. The withdrawal index can be set to an index outside of the group, resulting in incorrect totalDepositRoom accounting](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Stakelink

### Dates: Sep 30th, 2024 - Oct 17th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-09-stakelink)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 1
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. No LSTs transfer on node operator withdrawals resulting in stuck funds and loss for node operators            



## Summary

Inside `OperatorStakingPool`, node operators are required to stake their LSTs. Total LST staked and node operators balances are tracked by state variables:

```Solidity
// stores the LST share balance for each operator
mapping(address => uint256) private shareBalances;
// total number of LST shares staked in this pool
uint256 private totalShares;
```

Upon withdrawing (`OperatorStakingPool:_withdraw:L200-204`) `sharesBalances` and `totalShares` are updated but no LST is transfered from `OperatorStakingPool` back to operator. This leaves `OperatorStakingPool` with stuck LST tokens and node operators can't withdraw their stake.

```Solidity
function withdraw(uint256 _amount) external {
     if (!isOperator(msg.sender)) revert SenderNotAuthorized();
         _withdraw(msg.sender, _amount);
}

function _withdraw(address _operator, uint256 _amount) private {
      uint256 sharesAmount = lst.getSharesByStake(_amount);
 @>   shareBalances[_operator] -= sharesAmount;
 @>   totalShares -= sharesAmount;
  
      emit Withdraw(_operator, _amount, sharesAmount);
}
```

## Vulnerability Details

Vulnerable code: <https://github.com/Cyfrin/2024-09-stakelink/blob/main/contracts/linkStaking/OperatorStakingPool.sol#L199>

## PoC

```TypeScript
it('PoC:High:OperatorStakingPool.sol#L199-204=>No LSTs transfer on node operator withdrawals resulting in stuck funds', async () => {
    const { signers, accounts, opPool, lst } = await loadFixture(deployFixture)
    const operator = signers[0]
    const operatorDepositAmount = toEther(1000)

    // 1. ========= Deposit to operator staking pool =========

    // take snapshot of operator balance before deposit
    const operatorBalanceBeforeDeposit = await lst.balanceOf(operator.address)
    // take snapshot of operator staking pool balance before deposit
    const opStakingPoolBalanceBeforeDeposit = await lst.balanceOf(opPool.target)

    // deposit to operator staking pool
    await lst.connect(operator).transferAndCall(opPool.target, operatorDepositAmount, '0x')

    // take snapshot of operator balance after deposit
    const operatorBalanceAfterDeposit = await lst.balanceOf(operator.address)
    // take snapshot of operator staking pool balance after deposit
    const opStakingPoolBalanceAfterDeposit = await lst.balanceOf(opPool.target)

    // make sure operator balance decreased by the deposit amount
    assert.equal(operatorBalanceBeforeDeposit - operatorBalanceAfterDeposit, operatorDepositAmount)
    // make sure operator staking pool balance increased by the deposit amount
    assert.equal(opStakingPoolBalanceAfterDeposit, opStakingPoolBalanceBeforeDeposit + operatorDepositAmount)

    // 2. ========= Withdraw from operator staking pool =========
    
    // take snapshot of operator balance before withdraw
    const operatorBalanceBeforeWithdraw = await lst.balanceOf(operator.address)
    // take snapshot of operator staking pool balance before withdraw
    const opStakingPoolBalanceBeforeWithdraw = await lst.balanceOf(opPool.target)

    // withdraw from operator staking pool
    await opPool.connect(operator).withdraw(operatorDepositAmount)

    // take snapshot of operator balance after withdraw
    const operatorBalanceAfterWithdraw = await lst.balanceOf(operator.address)
    // take snapshot of operator staking pool balance after withdraw
    const opStakingPoolBalanceAfterWithdraw = await lst.balanceOf(opPool.target)

    // make sure operator principal is 0
    assert.equal(fromEther(await opPool.getOperatorPrincipal(accounts[0])), 0)
    // make sure operator staked is 0
    assert.equal(fromEther(await opPool.getOperatorStaked(accounts[0])), 0)

    // show that operator LST balance didn't change
    assert.equal(operatorBalanceAfterWithdraw, operatorBalanceBeforeWithdraw)
    // show that operator staking pool has the same balance as before the withdraw
    assert.equal(opStakingPoolBalanceAfterWithdraw, opStakingPoolBalanceBeforeWithdraw)
  })
```

**Running Poc:**

1. Copy test to `./test/linkStaking/operator-staking-pool.test.ts`
2. Run tests with `npx hardhat test ./test/linkStaking/operator-staking-pool.test.ts --network hardhat`

**Output:**

```Solidity
OperatorStakingPool
    ✔ PoC:High:OperatorStakingPool.sol#L200-204=>No LSTs transfer on operator withdrawals resulting in stuck funds (1499ms)
```

## Impact

**Likelihood: High**

This will happen on every call to `withdraw` function inside `OperatorStakingPool`.\
Also when owner calls `removeOperators` function which will call underlying withdraw method if operator has some stake.

**Impact: Medium**

Operators won't be able to retrieve their staked LSTs, and the funds will be temporarily locked inside the `OperatorStakingPool`. One way to handle this issue in production would be for the owner to upgrade this implementation with a new one that withdraws all stuck funds. By reviewing past `Withdraw` events, the owner could redistribute the funds back to the operators.

## Tools Used

Manual review, hardhat tests.

## Recommendations

Inside `_withdraw` method after totalShares update, send `amount` of LSTs back to `operator`.

```diff
+   import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
+   import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
    import "../core/interfaces/IStakingPool.sol";

    contract OperatorStakingPool is Initializable, UUPSUpgradeable, OwnableUpgradeable {
        using SafeERC20Upgradeable for IERC20Upgradeable;
+       using SafeERC20 for IERC20;
  
    ...
  
  /**
     * @notice Withdraws tokens
     * @param _operator address of operator with withdraw for
     * @param _amount amount to withdraw
     **/
    function _withdraw(address _operator, uint256 _amount) private {
        uint256 sharesAmount = lst.getSharesByStake(_amount);
        shareBalances[_operator] -= sharesAmount;
        totalShares -= sharesAmount;
+       IERC20(address(lst)).safeTransfer(_operator, _amount);

        emit Withdraw(_operator, _amount, sharesAmount);
    }
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Remove splitter will always revert if there are some rewards left on splitter contract            



## Summary

Stakelink users can register to share their LST rewards with other addresses through the `LSTRewardsSplitterController` contract. Upon registration, the user will set fee addresses, and an individual `LSTRewardSplitter` contract will be created for the user.

```Solidity
 function addSplitter(
        address _account,
        LSTRewardsSplitter.Fee[] memory _fees
    ) external onlyOwner {
        if (address(splitters[_account]) != address(0)) revert SplitterAlreadyExists();

        address splitter = address(new LSTRewardsSplitter(lst, _fees, owner()));
        splitters[_account] = ILSTRewardsSplitter(splitter);
   ...
```

All user-accrued rewards are distributed to fee addresses in the individual `LSTRewardsSplitter` contract inside the `_splitRewards` function.

```Solidity
   /**
     * @notice Splits new rewards
     * @param _rewardsAmount amount of new rewards
     */
    function _splitRewards(uint256 _rewardsAmount) private {
        for (uint256 i = 0; i < fees.length; ++i) {
            Fee memory fee = fees[i];
            uint256 amount = (_rewardsAmount * fee.basisPoints) / 10000;

            if (fee.receiver == address(lst)) {
                IStakingPool(address(lst)).burn(amount);
            } else {
                lst.safeTransfer(fee.receiver, amount);
            }
        }

        principalDeposits = lst.balanceOf(address(this));
        emit RewardsSplit(_rewardsAmount);
    }
```

When a user wants to remove a splitter, the `removeSplitter` function will be called on the `LSTRewardsSplitterController`. Upon removing the splitter, all rewards at that moment should be distributed to the fee addresses. However, if there are some rewards left, this will always fail because the controller contract will try to transfer the entire balance of the splitter contract to the user, including previously distributed rewards. This will revert because the balance is not reduced by the already distributed rewards.

```Solidity
/**
     * @notice Removes an account's splitter
     * @param _account address of account
     **/
    function removeSplitter(address _account) external onlyOwner {
        ILSTRewardsSplitter splitter = splitters[_account];
        if (address(splitter) == address(0)) revert SplitterNotFound();

        uint256 balance = IERC20(lst).balanceOf(address(splitter));
        uint256 principalDeposits = splitter.principalDeposits();
        if (balance != 0) {
            if (balance != principalDeposits) splitter.splitRewards();
@>          splitter.withdraw(balance, _account);
        }
```

## Vulnerability Details

Vulnerable code: <https://github.com/Cyfrin/2024-09-stakelink/blob/main/contracts/core/lstRewardsSplitter/LSTRewardsSplitterController.sol#L138>

## PoC

```Solidity
it('PoC:Medium:LSTRewardsSplitterController#134-138=>remove splitter will always revert if there are some rewards left on splitter contract', async () => {
    const { accounts, controller, token } = await loadFixture(deployFixture)

    // add new splitter for some account
    await controller.addSplitter(accounts[2], [
      { receiver: accounts[7], basisPoints: 4000 },
      { receiver: accounts[8], basisPoints: 4000 },
    ])

    // read newly created splitter
    const splitter = await ethers.getContractAt(
      'LSTRewardsSplitter',
      await controller.splitters(accounts[2])
    )

    // simulate reward amount
    const rewardAmount = toEther(100)
    await token.transfer(splitter.target, rewardAmount)
    
    // remove splitter will fail trying to transfer full balance amount without accounting for previous rewards distribution
    await expect(controller.removeSplitter(accounts[2])).to.be.reverted;
})
```

**Running Poc:**

1. Copy test to `./test/core/lst-rewards-splitter.test.ts`
2. Run tests with `npx hardhat test ./test/core/lst-rewards-splitter.test.ts --network hardhat`

**Output**

```Solidity
 LSTRewardsSplitter
    ✔ PoC:Medium:LSTRewardsSplitterController#134-138=>remove splitter does not work when there are some rewards left on splitter contract (1252ms)

```

## Impact

**Likelihood: Medium**

This will happen on every call to removeSplitter function inside LSTRewardsSplitterController if some rewards are accured.

**Impact: Medium**

The functionality of removing the splitter will be broken. The only way to remove the splitter would be to require that accrued rewards are zero. To ensure always correct functionality, the rewards would need to be distributed within the same block.

## Tools Used

Manual review, hardhat tests.

## Recommendations

After `splitRewards` call, only lst balance left on splitter contract should be transfered to user. This amount is captured at the end of `_splitRewardsFunction` inside `principalDeposits` state variable.\
&#x20;

```diff
/**
     * @notice Removes an account's splitter
     * @param _account address of account
     **/
    function removeSplitter(address _account) external onlyOwner {
        ILSTRewardsSplitter splitter = splitters[_account];
        if (address(splitter) == address(0)) revert SplitterNotFound();

        uint256 balance = IERC20(lst).balanceOf(address(splitter));
        uint256 principalDeposits = splitter.principalDeposits();
        if (balance != 0) {
            if (balance != principalDeposits) splitter.splitRewards();
-            splitter.withdraw(balance, _account);
+            splitter.withdraw(splitter.principalDeposits(), _account);
        }
```

A new test can be added to show that the functionality is now satisfied.

```Solidity
it('remove splitter should work when some rewards are left on splitter contract', async () => {
    const { accounts, controller, token } = await loadFixture(deployFixture)

    // add new splitter for accounts[2]
    await controller.addSplitter(accounts[2], [
      { receiver: accounts[7], basisPoints: 4000 },
      { receiver: accounts[8], basisPoints: 4000 },
    ])

    // read newly created splitter
    const splitter = await ethers.getContractAt(
      'LSTRewardsSplitter',
      await controller.splitters(accounts[2])
    )

    // simulate reward amount
    const rewardAmount = toEther(100)
    await token.transfer(splitter.target, rewardAmount)

    // take snapshot before removing splitter
    const firstReceiverBalanceBefore = await token.balanceOf(accounts[7])
    const secondReceiverBalanceBefore = await token.balanceOf(accounts[8])
    const splitterBalanceBefore = await token.balanceOf(splitter.target)
    const splitterPrincipalDepositsBefore = await splitter.principalDeposits()
    const accountBalanceBefore = await token.balanceOf(accounts[2])

    console.log('\n=================BEFORE REMOVE SPLITTER====================')
    console.log('firstReceiverBalanceBefore', firstReceiverBalanceBefore.toString())
    console.log('secondReceiverBalanceBefore', secondReceiverBalanceBefore.toString())
    console.log('splitterBalanceBefore', splitterBalanceBefore.toString())
    console.log('splitterPrincipalDepositsBefore', splitterPrincipalDepositsBefore.toString())
    console.log('accountBalanceBefore', accountBalanceBefore.toString())

    // make sure splitter has 'rewardAmount' to distribute
    assert.equal(splitterBalanceBefore - splitterPrincipalDepositsBefore, BigInt(rewardAmount))

    // remove splitter
    await controller.removeSplitter(accounts[2])

    const firstReceiverBalanceAfter = await token.balanceOf(accounts[7])
    const secondReceiverBalanceAfter = await token.balanceOf(accounts[8])
    const splitterBalanceAfter = await token.balanceOf(splitter.target)
    const accountBalanceAfter = await token.balanceOf(accounts[2])

    console.log('\n=================AFTER REMOVE SPLITTER====================')
    console.log('firstReceiverBalanceAfter', firstReceiverBalanceAfter.toString())
    console.log('secondReceiverBalanceAfter', secondReceiverBalanceAfter.toString())
    console.log('splitterBalanceAfter', splitterBalanceAfter.toString())
    console.log('accountBalanceAfter', accountBalanceAfter.toString())

    // show that rewards were distributed correctly and no funds were left on splitter contract
    assert.equal(firstReceiverBalanceAfter - firstReceiverBalanceBefore, BigInt(rewardAmount) * 4n / 10n)
    assert.equal(secondReceiverBalanceAfter - secondReceiverBalanceBefore, BigInt(rewardAmount) * 4n / 10n)
    assert.equal(splitterBalanceAfter, 0n)
    assert.equal(accountBalanceAfter, accountBalanceBefore + BigInt(rewardAmount) * 2n / 10n)
  })
```

Output:

```Solidity
LSTRewardsSplitter

=================BEFORE REMOVE SPLITTER====================
firstReceiverBalanceBefore 0
secondReceiverBalanceBefore 0
splitterBalanceBefore 100000000000000000000
splitterPrincipalDepositsBefore 0
accountBalanceBefore 10000000000000000000000

=================AFTER REMOVE SPLITTER====================
firstReceiverBalanceAfter 40000000000000000000
secondReceiverBalanceAfter 40000000000000000000
splitterBalanceAfter 0
accountBalanceAfter 10020000000000000000000
    ✔ remove splitter should work when some rewards are left on splitter contract (1286ms)


  1 passing (1s)
```



# Low Risk Findings

## <a id='L-01'></a>L-01. The withdrawal index can be set to an index outside of the group, resulting in incorrect totalDepositRoom accounting            



## Summary

Vaults are distributed into vault groups. Each vault group has a withdrawal index that represents the next vault in the group from which a withdrawal operation will be performed. Additionally, every group has a totalDepositRoom, representing how much can be deposited across all the vaults in the group.

```solidity
struct VaultGroup {
    // index of next vault in the group to be withdrawn from
    uint64 withdrawalIndex;
    // total deposit room across all vaults in the group
    uint128 totalDepositRoom;
}
```

Besides vault groups, there are vaults that do not need to be part of a group. The next such vault that accepts deposits is represented by the `depositIndex` inside the GlobalVaultState.

```Solidity
struct GlobalVaultState {
    // total number of groups
    uint64 numVaultGroups;
    // index of the current unbonded group
    uint64 curUnbondedVaultGroup;
    // index of next vault to receive deposits across all groups
    uint64 groupDepositIndex;
    // index of next non-group vault to receive deposits
@>  uint64 depositIndex;
}
```

When there is a new deposit, the `_depositToVaults` function will be called on the strategy, specifically the `VaultControllerStrategy`, which is inherited by both `OperatorVCS` and `CommunityVCS`. The new deposit is performed by passing a list of vaultIds. If one of the vaults is a withdrawal vault in its group and does not have any deposits, the group will be updated so that the `withdrawalIndex` points to the next vault in the group.&#x20;

```Solidity
/**
     * @notice Deposits tokens into vaults
     * @param _toDeposit amount to deposit
     * @param _minDeposits minimum amount of deposits that a vault can hold
     * @param _maxDeposits minimum amount of deposits that a vault can hold
     * @param _vaultIds list of vaults to deposit into
     */
    function _depositToVaults(
        uint256 _toDeposit,
        uint256 _minDeposits,
        uint256 _maxDeposits,
        uint64[] memory _vaultIds
    ) private returns (uint256) {
        
        ...

        // deposit into vaults in the order specified in _vaultIds
        for (uint256 i = 0; i < _vaultIds.length; ++i) {
            uint256 vaultIndex = _vaultIds[i];
            // vault must be a member of a group
            if (vaultIndex >= globalState.depositIndex) revert InvalidVaultIds();

            IVault vault = vaults[vaultIndex];
            uint256 groupIndex = vaultIndex % globalState.numVaultGroups;
            VaultGroup memory group = groups[groupIndex];
            uint256 deposits = vault.getPrincipalDeposits();
            uint256 canDeposit = _maxDeposits - deposits;

            globalState.groupDepositIndex = uint64(vaultIndex);

            // if vault is empty and equal to withdrawal index, increment withdrawal index to the next vault in the group
 @>           if (deposits == 0 && vaultIndex == group.withdrawalIndex) {
 @>               group.withdrawalIndex += uint64(globalState.numVaultGroups);
 @>               if (group.withdrawalIndex > globalState.depositIndex) {
 @>                   group.withdrawalIndex = uint64(groupIndex);
                }
            }
          
          ...
```

However, it is possible for the withdrawalIndex to become the depositIndex, meaning the next withdrawalIndex in the group could point to a vault that is not part of that group. This occurs because the withdrawalIndex only checks if the next index is greater than the depositIndex.

```Solidity
// if vault is empty and equal to withdrawal index, increment withdrawal index to the next vault in the group
            if (deposits == 0 && vaultIndex == group.withdrawalIndex) {
                group.withdrawalIndex += uint64(globalState.numVaultGroups);
   @>           if (group.withdrawalIndex > globalState.depositIndex) {
                    group.withdrawalIndex = uint64(groupIndex);
                }
            }
```

With the group’s withdrawalIndex set to the groupIndex, it is possible for this group to become the `curUnboundedVaultGroup`, meaning withdrawals will be possible from this group.

```Solidity
function updateVaultGroups(
        uint256[] calldata _curGroupVaultsToUnbond,
        uint256 _curGroupTotalDepositRoom,
        uint256 _nextGroup,
        uint256 _nextGroupTotalUnbonded
    ) external onlyFundFlowController {
        for (uint256 i = 0; i < _curGroupVaultsToUnbond.length; ++i) {
            vaults[_curGroupVaultsToUnbond[i]].unbond();
        }

        vaultGroups[globalVaultState.curUnbondedVaultGroup].totalDepositRoom = uint128(
            _curGroupTotalDepositRoom
        );
@>      globalVaultState.curUnbondedVaultGroup = uint64(_nextGroup);
        totalUnbonded = _nextGroupTotalUnbonded;
    }
```

With this group as the unbounded group, when a withdrawal is called inside `VaultControllerStrategy`, the `depositIndex` can be passed as the first `vaultId`. This index will be valid since it was previously set as the withdrawal index during a deposit.

```Solidity
    /**
     * @notice Withdraws tokens from vaults and sends them to staking pool
     * @dev called by VaultControllerStrategy using delegatecall
     * @param _amount amount to withdraw
     * @param _data encoded vault withdrawal order
     */
    function withdraw(uint256 _amount, bytes calldata _data) external {
        if (!fundFlowController.claimPeriodActive() || _amount > totalUnbonded)
            revert InsufficientTokensUnbonded();

        GlobalVaultState memory globalState = globalVaultState;
        uint64[] memory vaultIds = abi.decode(_data, (uint64[]));
        VaultGroup memory group = vaultGroups[globalState.curUnbondedVaultGroup];

        // withdrawals must continue with the vault they left off at during the previous call
@>      if (vaultIds[0] != group.withdrawalIndex) revert InvalidVaultIds();
```

If the vault with the depositIndex has some deposits and an active claim period for withdrawal, the withdrawal will go through this vault.

```Solidity
for (uint256 i = 0; i < vaultIds.length; ++i) {
            // vault must be a member of the current unbonded group
            if (vaultIds[i] % globalState.numVaultGroups != globalState.curUnbondedVaultGroup)
                revert InvalidVaultIds();

            group.withdrawalIndex = uint64(vaultIds[i]);
            IVault vault = vaults[vaultIds[i]];
            uint256 deposits = vault.getPrincipalDeposits();

@>           if (deposits != 0 && vault.claimPeriodActive() && !vault.isRemoved()) {
                if (toWithdraw > deposits) {
                    vault.withdraw(deposits);
                    unbondedRemaining -= deposits;
                    toWithdraw -= deposits;
                } else if (deposits - toWithdraw > 0 && deposits - toWithdraw < minDeposits) {
                    // cannot leave a vault with less than minimum deposits
                    vault.withdraw(deposits);
                    unbondedRemaining -= deposits;
                    break;
                } else {
                    vault.withdraw(toWithdraw);
                    unbondedRemaining -= toWithdraw;
                    break;
                }
            }
        }
```

The end result is that the totalDepositRoom will be increased by the withdrawn amount, even though part or all of that amount was withdrawn from a vault that is not part of that group. This will lead to inaccurate accounting, as the totalDepositRoom for this group will actually be lower than reported.

```Solidity
        totalDeposits -= totalWithdrawn;
        totalPrincipalDeposits -= totalWithdrawn;
        totalUnbonded = unbondedRemaining;

@>      group.totalDepositRoom += uint128(totalWithdrawn);
        vaultGroups[globalVaultState.curUnbondedVaultGroup] = group;
```

## Vulnerability Details

Vulnerable code: <https://github.com/Cyfrin/2024-09-stakelink/blob/main/contracts/linkStaking/base/VaultControllerStrategy.sol#L204>

Consider having 17 vaults:

```diff
 __. __. __. __. __. __. __. __. __. __. __. __. __. __. __. __. __
|__||__||__||__||__||__||__||__||__||__||__||__||__||__||__||__||__|
 0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15  16
```

Where first  16 vaults are parth of the 4 groups like this, and last vault is not part of any group.

```diff
 _________   _________   _________   _________
 | 0  | 4  | | 1  | 5  | | 2  | 6  | | 3  | 7  |
 |____|____| |____|____| |____|____| |____|____|
 | 8  | 12 | | 9  | 13 | |10  | 14 | |11  | 15 |
 |____|____| |____|____| |____|____| |____|____|  16
```

Here, the 16th vault is the `globalState.depositIndex`.\
During a deposit, if the **12th** vault is part of the vaultIds and it is the withdrawalIndex for group 0, the withdrawal index will be set to index 16. This is a valid index for group 0 because `16 % numOfGroups = 0`, but that vault is not part of the group.

```Solidity
if (deposits == 0 && vaultIndex == group.withdrawalIndex) {
@>  group.withdrawalIndex += uint64(globalState.numVaultGroups);
    if (group.withdrawalIndex > globalState.depositIndex) {
        group.withdrawalIndex = uint64(groupIndex);
    }
}
```

Later during withdraw phase, if group 0 becomes curUnboundedVaultGroup this check will pass:

```solidity
if (vaultIds[i] % globalState.numVaultGroups != globalState.curUnbondedVaultGroup)
                revert InvalidVaultIds();
```

This means the withdrawal will be executed from the 16th vault if possible, and an incorrect amount will be added to the depositRoom of group 0.

\
Impact
------

**Likelihood: Low**

Several conditions needs to be met for this to happen, making this low likelihood:

1. The vault with the withdrawal index needs to be empty during the deposit.
2. There must be a valid groupIndex that satisfies the condition `(withdrawalIndex + vault group size) = groupIndex`.
3. The group must become the `curUnboundedVaultGroup`.
4. The `groupIndex` vault must not be empty and should be in an active claim period.

**Impact: Medium**

The total depositRoom will be inaccurately updated for the group, which will lead to false assumptions and break functionality if the withdrawal amount is greater than the actual depositRoom for that group.

## Tools Used

Manual review.

## Recommendations

When checking for withdrawal index, use `>=` instead of `>` . This will make sure that withdrawalIndex is alway part of the group.

```diff
// if vault is empty and equal to withdrawal index, increment withdrawal index to the next vault in the group
            if (deposits == 0 && vaultIndex == group.withdrawalIndex) {
                group.withdrawalIndex += uint64(globalState.numVaultGroups);
-                if (group.withdrawalIndex > globalState.depositIndex) {
+                if (group.withdrawalIndex >= globalState.depositIndex) {
                    group.withdrawalIndex = uint64(groupIndex);
                }
            }
```




