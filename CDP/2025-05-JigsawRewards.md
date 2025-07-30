# Summary.

Jigsaw is a CDP-based stablecoin protocol that brings full flexibility and composability to your collateral through the concept of “dynamic collateral”. Jigsaw leverages crypto’s unique permissionless composability to enable dynamic collateral in a fully non-custodial way.


### Findings.

### [H-1]("https://cantina.xyz/code/7a40c849-0b35-4128-b084-d9a83fd533ea/findings/767") Liquidating users invested in Elixir strategy may be problematic.

The Elixir strategy differs from other staking protocols in that it uses a two-step withdrawal process:

1. The user initiates a withdrawal, which starts a cooldown timer:

```solidity
    function cooldownAssets(uint256 assets) external ensureCooldownOn returns (uint256 shares) {
       if (assets > maxWithdraw(msg.sender)) revert ExcessiveWithdrawAmount();

       shares = previewWithdraw(assets);

>>     cooldowns[msg.sender].cooldownEnd = uint104(block.timestamp) + cooldownDuration;
       cooldowns[msg.sender].underlyingAmount += uint152(assets);

       _withdraw(msg.sender, address(silo), msg.sender, assets, shares);
   }

```

2. After the `cooldown` period ends, the user can perform the actual unstake:

```solidity
    function unstake(address receiver) external {
       UserCooldown storage userCooldown = cooldowns[msg.sender];
       uint256 assets = userCooldown.underlyingAmount;

       if (block.timestamp >= userCooldown.cooldownEnd || cooldownDuration == 0) {
           userCooldown.cooldownEnd = 0;
           userCooldown.underlyingAmount = 0;

           silo.withdraw(receiver, assets);
       } else {
           revert InvalidCooldown();
       }
   }

```

Note that during the cooldown period, the user can initiate additional withdrawals, which resets the cooldown timer.

This withdrawal mechanism introduces serious complications for protocols that may need to liquidate users with positions in the Elixir strategy:

* A user may never call `ElixirStrategy.cooldown`, preventing any withdrawal — and therefore liquidation — from occurring. Even if the strategy owner initiates it on their behalf, the protocol must still wait 7 days (or however long the cooldown is), during which time the position may deteriorate and result in bad debt.
  
* Even if the user did call cooldown, there’s no guarantee they placed their entire shareholding into cooldown. In fact, a user can continuously “poke” the cooldown with a minimal share `(e.g., 1 wei)` to reset the cooldown timer indefinitely, effectively preventing liquidation.
  

## What went wrong and how to fix it next time.

* Didn't review the codebase like this.
* Dint't think about this attack vector.
  
## Take aways.

* When dealing with investment of collateral in strategies, always make sure that a user cannot brick liquidation by resetting the cooldown timers through for instance withdrawing their collateral partially.
* Make sure they are only full share not partial cooldowns during withdrawals.
* Make sure that anyone can put the shares of liquidatable users on cooldown, possibly with a small reward as incentive.
