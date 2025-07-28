# Summary.

The Mystic Finance making a environment dedicated to RWA-backed lending which takes RWA's different characteristics into account and not only supports them, but leverages them to their full potential.

### Findings.

### [H-1]("https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/findings/679") Missing access control in `stPlumeMinter::restake()` allows unauthorized fund restaking.

The `stPlumeMinter::restake()` lacks proper access control, allowing any external user to `restake` funds from cooling/parked pools to any validator. This operation should be restricted to accounts with the `REBALANCER_ROLE`.

```solidity
@>  function restake(uint16 validatorId) external nonReentrant returns (uint256 amountRestaked) { 
        _rebalance();
        (PlumeStakingStorage.StakeInfo memory info) = plumeStaking.stakeInfo(address(this));
        amountRestaked = plumeStaking.restake(validatorId, info.cooled + info.parked);
        
        emit Restaked(address(this), validatorId, amountRestaked);
        return amountRestaked;
    }
```
## What went wrong and how to fix it next time.

* Didn't really know that this is a bug.
  
## Take aways.

* Always make sure there is access control in the restake function.
  
### [H-2]("https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/findings/635") Unstaking extends the cooldown period for all queued withdrawals, causing DoS.

Users can withdraw funds by `unstaking` first then proceeding to call withdraw() once `cooldown` period has passed.

After unstaking, their funds are stored as `"parked"` (funds that have cooled and now can be withdrawn) and `"cooled"` (funds that have yet to be cooled down).

As `cooldown` period comes to an end, these funds convert from `"cooled"` to `"parked"`.

The problem is that any further unstaking call will extend the `cooldown` period:

```solidity
// 3. Set/Reset the cooldown timer for the current cooled balance
        globalInfo.cooldownEnd = block.timestamp + $.cooldownInterval;
```

So if this period is increased, the user who had queued for withdrawal previously has to now wait a longer period:

```solidity
function withdraw() external nonReentrant returns (uint256 amount) {
        PlumeStakingStorage.Layout storage $ = _getPlumeStorage();
        PlumeStakingStorage.StakeInfo storage info = $.stakeInfo[msg.sender];

        amount = info.parked;
@>      if (info.cooled > 0 && info.cooldownEnd <= block.timestamp) {
            uint256 cooledAmount = info.cooled; // Store before zeroing
            amount += cooledAmount;
            info.cooled = 0;

            // Need to adjust totalCooling if it exists and is accurate
            if ($.totalCooling >= cooledAmount) {
                $.totalCooling -= cooledAmount;
            } else {
                $.totalCooling = 0; // Avoid underflow
            }
            // Reset cooldown end time only if cooled amount becomes zero
            if (info.cooled == 0) {
                info.cooldownEnd = 0;
            }
        }
```

Therefore any `unstaking` calls that occur before the cooldown period ends, will continue to indefinitely `extend` the time period where users can actually withdraw their funds.

## What went wrong and how to fix it next time.

* Didn't really know that this is a bug.

## Take aways.

* Where the cooldown period has been `set`, always try and see whether it can be extended through `reentering` that same function that sets it if another function requires the current time to have surpassed the `cooldown` period in order to execute.


### [H-3]("https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/findings/406") `stPlumeMinter::withdraw()` disregards currentWithheldETH when `amount > currentWithheldETH`, leading to incorrect fund accounting.

The `withdraw()` method in the `stPlumeMinter` contract is responsible for allowing users to withdraw their previously unstaked native tokens. However, there’s a critical vulnerability in the logic used to manage `currentWithheldETH`. Specifically, when `amount > currentWithheldETH`, the contract calls `plumeStaking.withdraw()` to withdraw all unstaked tokens, but it resets currentWithheldETH to 0 without actually utilizing its value.

High-Impact Scenario:

1. currentWithheldETH = `0.01 Ether`
2. Alice `unstake()` 1 Ether and withdraw().
3. plumeStaking returns `1.09 Ether`. `1 Ether` is sent to Alice and currentWithheldETH = `0.09 Ether`.
4. Bob unstake() 10 Ether and withdraw().
5. plumeStaking returns 11 Ether. `10 Ether` is sent to Bob and currentWithheldETH = `1 Ether`.

This results in the protocol losing accounting.

```solidity
        if(amount > currentWithheldETH ){
            withdrawn = plumeStaking.withdraw();
@>            currentWithheldETH = 0;
        } else {
            withdrawn = amount;
            currentWithheldETH -= amount;
        }

```

## What went wrong and how to fix it next time.

* Didn't really think about this.

## Take aways.

When withdrawing, and `amount > witheldAmount`, make sure to check that the `withHeldAmount` is not being reset to zero but instead used to account towards the withdrawn amount.

### [H-4]("https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/findings/200") Staking of protocol fee funds (`withHoldEth`) leading to revenue loss.

The `stPlumeMinter` contains a bug that causes protocol fees `(tracked in withHoldEth)` to be unintentionally staked along with user funds. This occurs because the `_rebalance()` function calls depositEther(address(this).balance), which sends all available ETH to the staking contract regardless of whether some portion is reserved for protocol fees.

When `withdrawFee()` is later called to transfer accumulated fees to the protocol owner, the contract attempts to send the full `withHoldEth` amount using a low-level call. Since these fees have already been staked, the transfer will either fail completely or transfer an amount less than expected. The contract then unconditionally resets `withHoldEth` to 0, effectively "forgetting" about the fees that were not successfully transferred.

## What went wrong and how to fix it next time.

* Didn't really think about this.

## Take aways.

During `rebalancing` when ether is being sent to the staking contract make sure that the `withHeldEth` is not part of that amount because they would be problems in the `withdrawFee` function, when trying to withdraw the `withHeldEth` since there is non available.

### [H-5]("https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/findings/713") Incorrect Withdrawal Amount Accounting in `stPlumeMinter` Contract.

The `withdraw` function in the stPlumeMinter contract contains a critical accounting error that artificially inflates the withdrawn amount when the actual amount withdrawn from Plume staking is less than the requested amount. This can lead to users receiving more ETH than they should, potentially causing a `direct loss of funds for the protocol`.

In the `withdraw` function of the `stPlumeMinter` contract, there is a critical issue with how the withdrawn amount is calculated. When a user requests a withdrawal, after determining the amount to withdraw, there's a check that incorrectly updates the withdrawn amount:

```solidity
withdrawn = withdrawn>amount ? withdrawn :amount; //fees could be taken by staker contract so that less than requested amount is sent
```
This logic means that if the amount withdrawn from Plume staking `(withdrawn)` is less than the requested amount `(amount)`, the code actually sets withdrawn to be equal to the requested amount rather than the actual withdrawn amount. This is fundamentally incorrect accounting.

Later, the protocol fee is calculated based on this potentially inflated amount:

```solidity
uint256 cachedAmount = withdrawn>amount ? amount :withdrawn;
amount -= withholdFee;
withHoldEth += cachedAmount - amount;

```

## What went wrong and how to fix it next time.

* Didn't really think about this.


## Take aways.

* Always take a keen look at the if and else operators especially in the `withdraw` function.
* Pay attention to the logic while also doing partial withdrawals to see what breaks.

### [H-6]("https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/findings/91") DOS in `unstake` and `withdraw` due to zero contract balance in `_rebalance`.

The `_rebalance` function, which is called by `unstake` and `withdraw` operations, passes `address(this).balance` to the `depositEther` function. When all tokens are staked in PlumeStaking and the contract has no balance, these operations will fail until funds are added to the `stPlumeMinter` contract.

```solidity
function _rebalance() internal {
  ...
  
  depositEther(address(this).balance);  
}

function depositEther(uint256 _amount) internal returns (uint256 depositedAmount) {
  // Initial pause check
  require(!depositEtherPaused, "Depositing ETH is paused");

  require(_amount > 0, "Amount must be greater than 0");
  ...
}
```

## What went wrong and how to fix it next time.

* Didn't really think about this.


## Take aways.

* During rebalancing always makes sure there is a zero amount check before depositing to the staking contract.
* Always check the validity of state around other functions that depend on the `rebalance` function.


### [H-7]("https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/findings/89") Claiming causes double counting in `currentWithHeldEth`

The `_claim` function, called by `_rebalance`, invokes `PlumeStaking.claim` to retrieve earned rewards. However, there is an issue with how the stPlumeMinter handles these claimed rewards.

Let's examine how PlumeStaking.claim sends rewards to the `stPlumeMinter`


```solidity
function claim(
    address token
) external nonReentrant returns (uint256) {
    ...
    calculate totalRewards
    ...

    if (totalReward > 0) {
        // Transfer rewards from treasury to user
        _transferRewardFromTreasury(token, totalReward, msg.sender);

        emit RewardClaimed(msg.sender, token, totalReward);
    }
    return totalReward;
}
```

Since the tokens are sent from the treasury rather than the `PlumeStaking contract`, they trigger the receive function:

```solidity
receive() external payable override {
    if(msg.sender != address(plumeStaking)) {
        _submit(msg.sender);
    }
}
```

This additional call to `_submit` with the reward amount creates a double-counting issue in `currentWithheldEth`, which breaks withdrawal functionality by any user that wants to withdraw maximum of `currentWithheldEth`.


## What went wrong and how to fix it next time.

* Didn't really think about this.


## Take aways.

Always look at the `claim` function and follow what it does to the other contract in terms of storage updates.


### [H-8]("https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/findings/87") The `currentWithheldETH` incorrectly updates when unstake.

The `currentWithheldETH` incorrect updates when `unstaking` leading to incorrect withdrawn flow and funds might be stuck in `PlumeStaking`. Let’s take a look at the code and notes:

```solidity
 function unstake(uint256 amount) external nonReentrant returns (uint256 amountUnstaked) {
  ...
  // Check if we can cover this with withheld ETH
  if (currentWithheldETH >= amount) {
      amountUnstaked = amount;
      cooldownTimestamp = block.timestamp + 1 days;
  }else{
      uint256 remainingToUnstake = amount;
      amountUnstaked = 0;
      if (currentWithheldETH > 0) {
          amountUnstaked = currentWithheldETH;
          remainingToUnstake -= currentWithheldETH;
          currentWithheldETH = 0; 
      }
      ...
    }
    ...
  }

```

During the `withdraw` function, the `currentWithheldETH` is checked again in:

```solidity
function withdraw(address recipient) external nonReentrant returns (uint256 amount) {
	...
  if(amount > currentWithheldETH ){
      withdrawn = plumeStaking.withdraw();
      currentWithheldETH = 0;
  } else {
      withdrawn = amount;
      currentWithheldETH -= amount;
  }

```

When there is a two withdrawals less than `currentWithheldETH` , one of the withdrawals might get stuck until more deposits and withdrawals fix the `currentWithheldEth` accounting.


## What went wrong and how to fix it next time.

* Didn't really think about this.


## Take aways.

Pay attention to the `unstake ` function and make sure that the currentWithheldETH never gets set to zero.