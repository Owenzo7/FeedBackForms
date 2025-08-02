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

* When dealing with investment of collateral in strategies, always make sure that a user cannot brick liquidation by resetting the cooldown timers through for instance withdrawing the collateral partially.
* Make sure they are only full share not partial withdrawals during if it depends on cooldown functionality to avoid bricking liquidation.
* Make sure that anyone can put the shares of liquidatable users on cooldown, possibly with a small reward as incentive.
  

### [H-2]("https://cantina.xyz/code/7a40c849-0b35-4128-b084-d9a83fd533ea/findings/755") Partial withdrawals don't work with Elixir strategy.

The Elixir staking contract uses a two-step withdrawal process, where holders:

* Withdraw their shares from the vault, initiating a cooldown period.
* After the cooldown has passed, claim their assets.
  
This behavior is reflected in the `ElixirStrategy`. Initially, the holder calls a cooldown function to start the cooldown by specifying the number of shares:


```solidity
    function cooldown(address _recipient, uint256 _shares) external nonReentrant {
        require(
            msg.sender == owner() || msg.sender == IHoldingManager(manager.holdingManager()).holdingUser(_recipient),
            "1001"
        );

        _genericCall({
            _holding: _recipient,
            _contract: tokenOut,
            _call: abi.encodeCall(ISdeUsdMin.cooldownShares, _shares)
        });
    }
```

Once the cooldown period has passed, the user can initiate the general `withdraw` function. This function allows partial withdrawals and calculates a ratio to proportionally adjust both the `share` and `investment` values:

```solidity

        params.shareRatio = OperationsLib.getRatio({
            numerator: params.shares,
            denominator: params.totalShares,
            precision: params.shareDecimals,
            rounding: OperationsLib.Rounding.Floor
        });

        _burn({
            _receiptToken: receiptToken,
            _recipient: _recipient,
            _shares: params.shares,
            _totalShares: params.totalShares,
            _tokenDecimals: params.shareDecimals
        });

>>      params.investment = (recipients[_recipient].investedAmount * params.shareRatio) / 10 ** params.shareDecimals;
        uint256 deUsdBalanceBefore = IERC20(deUSD).balanceOf(address(this));

        _genericCall({
            _holding: _recipient,
            _contract: tokenOut,
            _call: abi.encodeCall(ISdeUsdMin.unstake, (address(this)))
        });

        uint256 deUsdAmount = IERC20(deUSD).balanceOf(address(this)) - deUsdBalanceBefore;

        // Swap deUSD to USDT on Uniswap
        _swapExactInputMultihop({
            _tokenIn: deUSD,
            _amountIn: deUsdAmount,
            _recipient: _recipient,
            _swapData: _data,
            _swapDirection: SwapDirection.ToTokenIn
        });

        // Take protocol's fee from generated yield if any.
        params.withdrawnAmount = IERC20(tokenIn).balanceOf(_recipient) - params.balanceBefore;
>>      params.yield = params.withdrawnAmount.toInt256() - params.investment.toInt256();

```

However, there's a flaw in this logic: the contract does not account for the number of `shares` that were actually placed on cooldown. Instead, it uses the number of shares specified at the time of the withdraw call. During the unstake operation, the full amount placed on cooldown will be returned, regardless of the smaller share input in the `withdrawal` call.

An attacker can exploit this as follows:

1. The user places all of their shares on cooldown.
2. Once the cooldown period is over, they call withdraw specifying only 1 wei of shares.
3. This results in a very small share ratio, making the investment calculated essentially zero.
4. However, the full amount placed on cooldown is withdrawn and recorded as yield, not principal.
5. This artificially doubles the user's collateral value at no cost.
   
## What went wrong and how to fix it next time.

* Didn't review the codebase like this.
* Dint't think about this attack vector.

## Take aways.

* While doing partial withdrawals try and see if their is a way to inflate collateral with yield that its not suppose to have (`unfair amount of yield`). i.e before withdrawals, a cooldown function is provided and lets say that you want all the sahres withdrawn and then in the actual withdrawn function it doesn't use the provided shares you declared in the cooldown function thus breaking some functionality which makes you bag a high and unfair amount of yield.

### [H-3]("https://cantina.xyz/code/7a40c849-0b35-4128-b084-d9a83fd533ea/findings/50") Swap Path Manipulation Leads to Funds Theft During a Dust-sized Liquidations.

Swap path manipulation during ElixirStrategy.withdraw allows user funds theft and `performance fee` avoidance.

During a liquidation, a liquidator can provide strategies to withdraw collateral from:

```solidity
        // If strategies are provided, retrieve collateral from strategies if needed.
        if (_data.strategies.length > 0) {
            _retrieveCollateral({
                _token: _collateral,
                _holding: holding,
                _amount: collateralUsed,
                _strategies: _data.strategies,
                _strategiesData: _data.strategiesData,
                useHoldingBalance: true
            });
        }
```

During such retrieval, the collateral invested in one strategy is retrieved in full, regardless of the amount needed for the liquidation:

```javascript
        // Iterate over sent strategies and retrieve collateral.
        for (uint256 i = 0; i < _strategies.length; i++) {
            (, tempData.shares) = IStrategy(_strategies[i]).recipients(_holding);

            // Withdraw collateral.
            (tempData.withdrawResult,,,) = _getStrategyManager().claimInvestment({
                _holding: _holding,
                _token: _token,
                _strategy: _strategies[i],
                _shares: tempData.shares,
                _data: _strategiesData[i]
            });

```

That means that even if `_jUsdAmount` liquidated is just some wei-s, the full strategy withdrawal (possibly significant amount) is triggered.

Besides, the liquidator can provide arguments for collateral retrieval via `_data.strategiesData`. In the case of ElixirStrategy this argument is eventually passed as the _data argument of `withdraw()`. The data contains parameters to the `_swapExactInputMultihop` function.

```javascript

    function _swapExactInputMultihop(
        address _tokenIn,
        uint256 _amountIn,
        address _recipient,
        bytes calldata _swapData,
        SwapDirection _swapDirection
    ) private returns (uint256 amountOut) {
        // Decode the data to get the swap path
        (uint256 amountOutMinimum, uint256 deadline, bytes memory swapPath) =
            abi.decode(_swapData, (uint256, uint256, bytes));

        // Validate swap path length
        // Minimum path length is 43 bytes (length of smallest encoded pool key = address[20] + fee[3] + address[20])
        if (swapPath.length < 43) revert InvalidSwapPathLength();

        // Validate token path integrity for FromTokenIn direction:
        // - First token must be tokenIn
        // - Last token must be `deUSD` that's later used for staking
        if (_swapDirection == SwapDirection.FromTokenIn) {
            if (swapPath.toAddress(0) != tokenIn) revert InvalidFirstTokenInPath();
            if (swapPath.toAddress(swapPath.length - ADDR_SIZE) != deUSD) revert InvalidLastTokenInPath();
        }

        // Validate token path integrity for ToTokenIn direction:
        // - First token must be tokenOut
        // - Last token must be `deUSD` that's later used for unstaking
        if (_swapDirection == SwapDirection.ToTokenIn) {
            if (swapPath.toAddress(0) != deUSD) revert InvalidFirstTokenInPath();
            if (swapPath.toAddress(swapPath.length - ADDR_SIZE) != tokenIn) revert InvalidLastTokenInPath();
        }

        // Validate amountOutMin is within allowed slippage
        if (amountOutMinimum < getAllowedAmountOutMin(_amountIn, _swapDirection)) revert InvalidAmountOutMin();

        // Approve the router to spend `_tokenIn`.
        IERC20(_tokenIn).forceApprove({ spender: uniswapRouter, value: _amountIn });

        // A path is a  encoded as (tokenIn, fee, tokenOut/tokenIn, fee, tokenOut).
        ISwapRouter.ExactInputParams memory params = ISwapRouter.ExactInputParams({
            path: swapPath,
            recipient: _recipient,
            deadline: deadline,
            amountIn: _amountIn,
            amountOutMinimum: amountOutMinimum
        });
```

The parameter swapPath contains (possibly multihop) a swap path to execute. Note that an attacker has significant control over that path: the only validation is the first and the last tokens. This means that the attacker can use their controlled pools inside the path, and perform a swap like:

```md
deUSD -> USDT -> HACK_TOKEN -> USDT -> ... -> HACK_TOKEN -> USDT -> tokenIn
```

The attacker can have full control over their pools (as he's the only one having HACK_TOKEN), and can set any Uniswap-supported swap fees in the pools. This allows the attacker to drain user collateral up to the allowedSlippagePercentage.

To sum up, the attacker can initiate minimal liquidation (e.g. for 0.01 jUSD) of a liquidatable position, properly liquidate such a tiny portion, but as a side effect convert possibly a significant portion of user collateral using attacker-controlled pools and steal a large portion of user collateral (up to allowedSlippagePercentage). Such an attack may be executed against any liquidatable position.

As a result, there is a collateral theft and impact on the CDP collateralization ratio, possibly resulting in bad debt.

A user may use a similar technique during their withdrawal to create a fake negative yield and avoid paying a performance fee: swap using controlled pools, extract yield in the form of pool fees, and recognize a minimal loss investment, avoiding the performance fee.

## What went wrong and how to fix it next time.

* Didn't review the codebase like this.
* Dint't think about this attack vector.

## Take aways.

* When dealing with swap paths in a swap function always check to see if there is only validation for the first and last tokens. If its the only validation then thats a bug.


### [H-4]("https://cantina.xyz/code/7a40c849-0b35-4128-b084-d9a83fd533ea/findings/189") Misplaced liquidatable check allows healthy positions to be liquidated.

`LiquidationManager::liquidate()` only allows liquidatable holdings to be liquidated.

However, the `isLiquidatable()` require is misplaced, which will allow the liquidation of healthy positions.

A misplaced require on `LiquidationManager::liquidate()` makes it so that the changes to invested collateral will not be tracked at all for liquidation, even after the collateral is retrieved:

```solidity
    function liquidate(
        ...
@1>     require(stablesManager.isLiquidatable({ _token: _collateral, _holding: holding }), "3073");
        ...
        // If strategies are provided, retrieve collateral from strategies if needed.
        if (_data.strategies.length > 0) {
@2>         _retrieveCollateral({
```

**@1:** Upon liquidation, the function requires the user to be liquidatable right in the beggining of the function.
**@2:** Then, the function retrieves collateral from the strategies.

Retrieving the collateral tracks the changes accrued while the collateral was invested. However, since the `isLiquidatable()` check was already done, the changes will remain untracked.

## What went wrong and how to fix it next time.

* Didn't review the codebase like this.
* Dint't think about this attack vector.
  
## Take aways.

* Check to see if healthy positions can be liquidated.
* Check to see if the `isLiquidatable` function gives the status of positions accurately.


### [H-5]("https://cantina.xyz/code/7a40c849-0b35-4128-b084-d9a83fd533ea/findings/158") Misplaced liquidatable check allows healthy positions to be liquidated.

A flaw in the `liquidation` routines prevents `under-collateralized` positions from being liquidated when collateral is held in yield strategies and the strategy has experienced a negative yield. Specifically, if pulling collateral during liquidation triggers a loss, the subsequent StablesManager.removeCollateral call reverts on the isSolvent check, aborting the entire liquidation. An attacker can exploit this to mint unbacked jUSD and generate bad debt that cannot be repaid through standard liquidation, leading to a direct loss to the protocol.

There is an issue in the liquidation routines, namely LiquidationManager.liquidate and LiquidationManager.liquidateBadDebt.

When a holding keeps its collateral in strategies, the protocol must pull the collateral out before liquidation:

```solidity

        // If strategies are provided, retrieve collateral from strategies if needed.
        if (_data.strategies.length > 0) {
            _retrieveCollateral({
                _token: _collateral,
                _holding: holding,
                _amount: collateralUsed,
                _strategies: _data.strategies,
                _strategiesData: _data.strategiesData,
                useHoldingBalance: true
            });
        }

```

The `_retrieveCollateral` boils down to `_getStrategyManager().claimInvestment` calls:

```solidity

        // Iterate over sent strategies and retrieve collateral.
        for (uint256 i = 0; i < _strategies.length; i++) {
            (, tempData.shares) = IStrategy(_strategies[i]).recipients(_holding);

            // Withdraw collateral.
            (tempData.withdrawResult,,,) = _getStrategyManager().claimInvestment({
                _holding: _holding,
                _token: _token,
                _strategy: _strategies[i],
                _shares: tempData.shares,
                _data: _strategiesData[i]
            });

```

The `_retrieveCollateral` method calls `StrategyManager.claimInvestment`, which handles both positive and negative yields:

```solidity

        (tempData.withdrawnAmount, tempData.initialInvestment, tempData.yield, tempData.fee) =
            tempData.strategyContract.withdraw({ _shares: _shares, _recipient: _holding, _asset: _token, _data: _data });
        require(tempData.withdrawnAmount > 0, "3016");

        if (tempData.yield > 0) {
            _getStablesManager().addCollateral({ _holding: _holding, _token: _token, _amount: uint256(tempData.yield) });
        }
        if (tempData.yield < 0) {
            _getStablesManager().removeCollateral({ _holding: _holding, _token: _token, _amount: tempData.yield.abs() });   // <-----
        }
```

To sum up, the call stack is the following:

```md
    LiquidationManager.liquidate | LiquidationManager.liquidateBadDebt -> StrategyManager.claimInvestment -> StablesManager.removeCollateral
``

On negative yield, `StablesManager.removeCollateral` unregisters collateral then asserts `isSolvent`, causing a revert during liquidation. Thus, any position with a strategy loss is unliquidatable, allowing minting of jUSD without adequate backing. An attacker can deposit collateral, incur a strategy loss, borrow jUSD, and then avoid liquidation indefinitely, extracting value and leaving the protocol with bad debt.

An attack vector may be proposed:
    1. Create a holding, StrategyManager.invest all collateral in a strategy that may experience a negative yield event. The more volatile the strategy and collateral price the better (non-stable collateral like Ether works best). Do not borrow much yet.
    2. Wait for a negative yield event. The amplitude of the negative yield is not that important. Important only the expected time of the loss recuperation.
    3. Optionally, add more collateral to the same strategy to amplify the attack.
    4. Borrow as much as possible.
    5. At this moment the attacker has a bullet-proof position: on the collateral appreciation, they pocket trading profits. On collateral depreciation, the position cannot be liquidated. Below the bad debt level, the attacker may buy back the collateral amount in a market and pocket excess debt (the excess debt in the attacker's hands is positive since the collateral value is now less than the debt minted). The net result for the attacker is the excess debt amount. The net result for the protocol is a loss in the form of bad debt (in essence distributed among jUSD holders).


## What went wrong and how to fix it next time.

* Didn't review the codebase like this.
* Dint't think about this attack vector.

## Take aways.

* Check to see if the position of the isSolvent check in a function bricking the liquidation process.
* Check to see if a position that has negative yield cannot be liquidated because of mainly because of some check like for instance isSolvent check that's been positioned badly.


### [H-6]("https://cantina.xyz/code/7a40c849-0b35-4128-b084-d9a83fd533ea/findings/98") ElixirStrategy wrong withdrawal accounting allows undercollateralized borrowing.

The ElixirStrategy contract incorrectly calculates withdrawn amounts during the withdrawal process by not properly tracking the initial token balance, which allows users to artificially inflate their collateral values and borrow significantly more tokens than they should be able to.

In the `ElixirStrategy` contract, when users withdraw their assets, the contract incorrectly calculates the withdrawn amount by using the following logic:

```solidity
params.withdrawnAmount = IERC20(tokenIn).balanceOf(_recipient) - params.balanceBefore;
```

The issue is that `params.balanceBefore` is 0 instead of capturing the actual token balance before the withdrawal process starts. This means if a user's holding already contains tokens of the same type `(tokenIn)`, all of those pre-existing tokens will be counted as "withdrawn" from the strategy during the withdrawal calculation.

When the protocol calculates yield from a strategy withdrawal, it uses this incorrectly inflated `withdrawnAmount` value, which causes the `SharesRegistry` to add a much larger amount of collateral to the user's holding than it should. This breaks a key security guarantee of the protocol by allowing users to borrow against collateral they don't actually have.

The attack path is as follows:
    1. User deposits a large amount of tokens (e.g., 100,000e6 USDT) into their holding
    2. User invests a minimal amount (e.g., 1e6 USDT) into the ElixirStrategy
    3. User withdraws all shares from the strategy
    4. During withdrawal, withdrawnAmount is calculated as the entire holding balance (original deposit + actual withdrawal), rather than just the actual withdrawal amount
    5. The registry adds this inflated value as collateral to the user's holding, thinking it's yield from the strategy
    6. User can now borrow against this artificially inflated collateral value

## What went wrong and how to fix it next time.

* Didn't review the codebase like this.
* Dint't think about this attack vector.
  
## Take aways.

* Check to see if the intial token balance is being tracked before withdrawals to avoid withdrawing amounts that are super-inflated.

### [H-6]("https://cantina.xyz/code/7a40c849-0b35-4128-b084-d9a83fd533ea/findings/6") LiquidationManager::liquidate() allows liquidators to get liquidation bonus and leave bad debt behind.

`LiquidationManager::liquidate()` allows partial liquidations, and doesn't check if the liquidated user is solvent.

This allows liquidators to liquidate only up to the user's full collateral, leaving them with only bad debt and no collateral.

```solidity
    function liquidate(
        …
@1>     require(_jUsdAmount <= ISharesRegistry(registryAddress).borrowed(holding), "2003");
@2>     require(stablesManager.isLiquidatable({ _token: _collateral, _holding: holding }), "3073");
        …
        // Remove collateral from holding.
@3>     stablesManager.forceRemoveCollateral({ _holding: holding, _token: _collateral, _amount: collateralUsed });

```

@1: The function allows the liquidator to liquidate any amount, as long as it is less than or equal to the user's debt
@2: The function requires the user to be `liquidatable`
@3: After liquidation, the function removes the liquidated collateral via StablesManager::forceRemoveCollateral() which skips the `solvency check`

This means if the liquidated user has bad debt, the liquidator will be able to take all of their existing collateral and still get the liquidation bonus, while leaving behind all bad debt.

## What went wrong and how to fix it next time.

* Didn't review the codebase like this.
* Dint't think about this attack vector.


## Take aways.

* always check to see if the position of the user worsens during partial liquidations whereby a liquidator can bag a liquidator bonus from the collateral left behind thus making the position to be in bad debt since it will be the `collateral - liquidatorBonus`.

### [H-7]("https://cantina.xyz/code/7a40c849-0b35-4128-b084-d9a83fd533ea/findings/5") `LiquidationManager::liquidateBadDebt()` will not retrieve all invested collateral from liquidated holding.

When the admin liquidates bad debt, they might not get all of the collateral from the user, further increasing the losses introduced by the bad debt.

`LiquidationManager::liquidateBadDebt()` attempts to retrieve `totalCollateral = SharesRegistry::collateral[holding]:`

```solidity
        uint256 totalCollateral = ISharesRegistry(registryAddress).collateral(holding);

        // If strategies are provided, retrieve collateral from strategies if needed.
        if (_data.strategies.length > 0) {
            _retrieveCollateral({
                _token: _collateral,
                _holding: holding,
@1>             _amount: totalCollateral,
                _strategies: _data.strategies,
                _strategiesData: _data.strategiesData,
                useHoldingBalance: true

```

`_retrieveCollateral()` stops retrieving collateral from strategies when the holding's balance meets the `_amount` (passed as totalCollateral), regardless of whether or not it has looped through all of the strategies passed:

```solidity
@2>         if (useHoldingBalance && IERC20(_token).balanceOf(_holding) >= _amount) break;
```

The issue is that the collateral on the shares registry does not register the yield accrued on external strategies.

So if the holding has, for example:

  * Deposited 100e18 to strategy 1, which increased to 110e18.
  * Deposited 5e18 to strategy 2, which increased to 10e18.
The function will attempt to retrieve a total of 105e18, which is the deposited collateral. After the first loop, the holding balance will be 110e18, which satisfies condition @1 above.

The loop will break and will not retrieve the collateral from strategy 2. The admin will lose 10e18, which is the current value deposited to strategy 2.

## What went wrong and how to fix it next time.

* Didn't review the codebase like this.
* Dint't think about this attack vector.

## Take aways.

* When dealing with collateral thats been invested in yield strategies, make sure during liquidation that `collateral + yield` is attributed to the liquidator instead of the `collateral` only.


### [H-8]("https://cantina.xyz/code/7a40c849-0b35-4128-b084-d9a83fd533ea/findings/2") If jUSD depegs, it will get in a death spiral.

The jUSD amount to mint upon `HoldingManager::borrow()` is calculated by querying the oracle price of jUSD.

So if jUSD depegs, users will be able to mint it for a discounted price, which means the collateral required for each jUSD will potentially be worth less than USD 1, which will further increase the depeg effect.

```solidity

    function borrow(
        …
        tempData.amount = _transformTo18Decimals({ _amount: _amount, _decimals: IERC20Metadata(_token).decimals() });

        // Get the USD value for the provided collateral amount.
        tempData.amountValue =
            tempData.amount.mulDiv(tempData.registry.getExchangeRate(), tempData.exchangeRatePrecision);

        // Get the jUSD amount based on the provided collateral's USD value.
@>      jUsdMintAmount = tempData.amountValue.mulDiv(tempData.exchangeRatePrecision, manager.getJUsdExchangeRate());
```

The line marked above converts the USD value of the collateral by the jUSD exchange rate.

So if jUSD depegs, `manager.getJUsdExchangeRate(`) will return a value smaller than tempData.exchangeRatePrecision, which is hardcoded in Manager.sol as 1e18.

So if a user wants to mint jUSD based on USD `1,000` worth of collateral, jUsdMintAmount will evaluate to:

(Assuming jUSD depegs to USDC 0.90)
`1,000 * 1e18 / 0.9e18 = 1,111`

## What went wrong and how to fix it next time.

* Didn't review the codebase like this.
* Dint't think about this attack vector.


## Take aways.

* If the stablecoin to mint depends on querying an oracle price then its obvious for depegging to happen. (meaning that minting can happen for less than `1USD`)

### [M-1]("https://cantina.xyz/code/7a40c849-0b35-4128-b084-d9a83fd533ea/findings/586") Malicious User Can Game Protocol Liquidation via Direct Deposit.

Direct collateral transfers to user holdings aren’t tracked, enabling users to leverage unaccounted collateral in strategies. This can result in bad debt after strategy losses, causing the protocol owner to absorb losses and allowing potential protocol manipulation.

Users can transfer collateral tokens directly to their holding addresses without this being tracked in the protocol’s collateral accounting. The protocol only updates collateral balances on explicit deposit calls, so these direct transfers remain untracked but can still be invested in strategies.

When a user invests this untracked collateral, any strategy losses reduce the actual collateral value. However, since the protocol’s accounting doesn’t reflect the extra collateral, even a small strategy loss can immediately push the user’s position into bad debt.

As a result, the protocol owner is forced to liquidate the user’s position, effectively covering the outstanding debt. This creates an exploitable scenario where users can game the system by leveraging untracked collateral: if the strategy wins, the user profits normally; if it loses, the user’s loss is minimized because they keep previously minted jUsd while the owner absorbs the bad debt.

Note: Untracked collateral is never sent to the owner during liquidation. Additionally, this untracked collateral is withdrawable by the user using `HoldingManager.sol#withdraw()`.


## What went wrong and how to fix it next time.

* Didn't review the codebase like this.
* Dint't think about this attack vector.

## Take aways.

* Check to see if a user can game the protocol through direct donation to his or her balance if its not tracked by the protocol during liquidation.
* When liquidating positions that have invested in strategies and it tanks below the value of the debt, make sure the `tracked + untracked` collateral value is accounted for to the liquidator during the liquidation stage.

### [M-2]("https://cantina.xyz/code/7a40c849-0b35-4128-b084-d9a83fd533ea/findings/1436") Collateral can not be properly retired by making its sharesRegistry inactive

The protocol has a feature for making a whitelisted collateral's registry inactive when they plan to end support for it. This is part of the protocol lifecycle.

Intended sequence :

* Collateral is whitelisted and its sharesRegistry is added to StablesManager contract.
* Right now the registry is set to active in StablesManager, with the ability to set it to inactive by the protocol admins when they want to retire the collateral product/ end support for it in Jigsaw.
* The reason for sunsetting a collateral could be anything : protocol economic risks, business decision and partnerships, collateral becomes malicious or hacked
* A major risk would be the depegging of other stablecoins supported as collateral in Jigsaw
This is reasonable given that some collateral tokens supported in Jigsaw have deppeged in the recent past (USD0PP and USDC

* But the problem is that the collateral retirement will not be possible because of the way isRegistryActive checks are applied across the whole codebase.

For example :

* `liquidateBadDebt()` and `liquidate()` => `forceRemoveCollateral()` reverts if registry is inactive so it will block bad debt liquidations
* `StrategyManager.claimInvestment()` => `forceRemoveCollateral()/ addCollateral()` it checks that collateral registry is active so withdrawals from strategy will again be impossible.
* `repay() => check isRegistryActive`
* `withdraw() => removeCollateral()` checks the registry should be active.
  
All these methods to `exit/ withdraw funds` from the protocol should still be accessible after the collateral is retired, so that pending bad debt positions can be closed and honest depositors can acquire their collateral. If the registry is ever set to inactive, then it will lock all user funds as well as repayment mechanisms, making `JUSD` worthless because of the potential bad debt.

Note that this collateral retirement can also not be done via other controls like `iswhitelisted`, sharesRegistry set to zero or other checks because every check blocks some of these operations. Not even pausing the contracts will help because it will pause everything.

Hence, there is no way to disable borrowing on a specific collateral if it gets depegged or hacked or anything because if the sharesRegistry is disabled it will also disable liquidations and repayments etc.

## What went wrong and how to fix it next time.

* Didn't review the codebase like this.
* Dint't think about this attack vector.
  
## Take aways.

* Make sure when a collateral token is blacklisted, inactive or even disabled it should be such that it only `blocks new deposits/ borrows and strategy investments` etc. but does not block `exits withdrawals/ repayments and liquidations` of the existing positions in the system.


### [M-3]("https://cantina.xyz/code/7a40c849-0b35-4128-b084-d9a83fd533ea/findings/1374") Withdrawals of rewards earned through strategies should not be charged with withdrawal fees.

f we check the StrategyManager.sol logic, we can see that any claimRewards() call by holding user will lead to strategy.claimRewards(). For example, Aave deposits give extra reward tokens, so the strategy helps us withdraw these from the Aave rewardController.

Here is the code from AaveV3StrategyV2.sol :

```solidity
    function claimRewards(
        address _recipient,
        bytes calldata
    ) external override nonReentrant onlyStrategyManager returns (uint256[] memory, address[] memory) {
        // aTokens should be checked for rewards eligibility.
        address[] memory eligibleTokens = new address[](1);
        eligibleTokens[0] = tokenOut;

        // Make the claimAllRewards through the user's Holding.
        (, bytes memory returnData) = _genericCall({
            _holding: _recipient,
            _contract: address(rewardsController),
            _call: abi.encodeCall(IRewardsController.claimAllRewards, (eligibleTokens, _recipient))
        });
```

This will ultimately lead to the reward tokens being forwarded to the holding address.

Now this `claimRewards()` logic also charges a performance fee on the rewards, because these are earned via jigsaw.

```solidity

        (uint256 performanceFee,,) = _getStrategyManager().strategyInfo(address(this));
        address feeAddr = manager.feeAddress();

        // Take performance fee for all the rewards.
        for (uint256 i = 0; i < rewardsList.length; i++) {
            uint256 fee = OperationsLib.getFeeAbsolute(claimedAmounts[i], performanceFee);
            if (fee > 0) {
                claimedAmounts[i] -= fee;
                emit FeeTaken(rewardsList[i], feeAddr, fee);
                IHolding(_recipient).transfer({ _token: rewardsList[i], _to: feeAddr, _amount: fee });
            }
        }

```

These performance fees are directly taken from the holding. The `MAX_PERFORMANCE_FEE` as per the manager contract is upto 25 %.

So upto `25 %` of fees has been already deducted form the rewards earned.

Then how will the holding owner claim these rewards out of the system for his own external use ? He will have to use the `withdraw()` function in `HoldingManager`.

The problem is that the `withdraw()` function again charges a withdrawal fee on these rewards as well.

The `MAX_WITHDRAWAL_FEE` as per manager contract is `8 %`, meaning the rewards earned would be reduced again by upto `8 %`.

This is unnecessary because the withdrawal fee is supposed to be for users exiting the system by taking out the collateral, while the rewards earned should be solely belonging to the holding owner, as the protocol has already taken upto a massive `25 %` cut.

This will lead to even lower yields when users take these rewards out of the jigsaw system.


## What went wrong and how to fix it next time.

* Didn't review the codebase like this.
* Dint't think about this attack vector.
  

## Take aways.

* When claiming rewards from the system and then withdrawing those same `fees` out of the system for external use, make sure there is `double fee` applied only from one part of the functionality like just for claiming rewards and not on the withdrawal functionality.