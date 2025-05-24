# Alchemix Transmuter - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Locked WETH in strategies can't be swapped and will lead to inaccurate calculations of total assets and fee distributions during the harvest and report call.](#M-01)
- ## Low Risk Findings
    - ### [L-01. Hanging approvals on old router contracts can lead to the loss of funds if routers are compromised.](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Alchemix

### Dates: Dec 16th, 2024 - Dec 23rd, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-12-alchemix)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 1
- Low: 1



    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Locked WETH in strategies can't be swapped and will lead to inaccurate calculations of total assets and fee distributions during the harvest and report call.            



## Summary

Keepers have the ability to call a function to harvest and record all profits accrued or losses incurred within the strategy. This function is utilized in the `TokenizedStrategy` implementation contract on Yearn Finance:

<https://github.com/yearn/tokenized-strategy/blob/master/src/TokenizedStrategy.sol#L1081>

```Solidity
function report()
        external
        nonReentrant
        onlyKeepers
        returns (uint256 profit, uint256 loss)
    {
        // Cache storage pointer since its used repeatedly.
        StrategyData storage S = _strategyStorage();

        // Tell the strategy to report the real total assets it has.
        // It should do all reward selling and redepositing now and
        // account for deployed and loose `asset` so we can accurately
        // account for all funds including those potentially airdropped
        // and then have any profits immediately locked.
@=>     uint256 newTotalAssets = IBaseStrategy(address(this))
            .harvestAndReport();

      ....
    }

```

During this call, specific code in the `BaseStrategy` will be executed to retrieve the total assets currently held:

```Solidity
    function harvestAndReport() external virtual onlySelf returns (uint256) {
@=>      return _harvestAndReport();
    }
```

All three strategies override the `_harvestAndReport` call with the same logic.

<https://github.com/Cyfrin/2024-12-alchemix/blob/main/src/StrategyMainnet.sol#L172>

```Solidity
function _harvestAndReport()
        internal
        override
        returns (uint256 _totalAssets)
    {
        ...
        uint256 unexchanged = transmuter.getUnexchangedBalance(address(this));
        // NOTE : possible some dormant WETH that isn't swapped yet (although we can restrict to only claim & swap in one tx)
@=>     uint256 underlyingBalance = underlying.balanceOf(address(this));
        _totalAssets = unexchanged + asset.balanceOf(address(this)) + underlyingBalance;
    }
```

The main purpose of this function is to return an accurate count of alETH held by the strategy along with its idle tokens, enabling the report call within the strategy to accurately calculate new fees and assets. This function also includes the balance of WETH that has not yet been swapped, as noted in the comment. However, this dormant WETH won't be swappable since the only place where WETH is swapped is during the `claimAndSwap` call, which takes WETH from the `Transumer` contract, not directly from the strategy contract itself.

```Solidity
function claimAndSwap(
    uint256 _amountClaim, 
    uint256 _minOut, 
    uint256 _routeNumber
) external onlyKeepers {
@=>  transmuter.claim(_amountClaim, address(this));
  ...
}
```

This will lead to inaccurate totalAssets reported by the `_harvestAndReport()` function because it includes the WETH balance that can't be swapped for alETH. This miscalculation will be used by the report function, leading to consequences like:

* Virtually increased profit and minting shares for fee recipients for assets not actually held by the strategy.
* Invalid totalAssets updates that do not reflect the real, available amount of alETH.

```Solidity
function report()
        external
        nonReentrant
        onlyKeepers
        returns (uint256 profit, uint256 loss)
    {
        ...

        uint256 newTotalAssets = IBaseStrategy(address(this))
            .harvestAndReport();

        uint256 oldTotalAssets = _totalAssets(S);
        
       // Calculate profit/loss.
        if (newTotalAssets > oldTotalAssets) {
            // We have a profit.
            unchecked {
                profit = newTotalAssets - oldTotalAssets;
            }

            // We need to get the equivalent amount of shares
            // at the current PPS before any minting or burning.
@=>         sharesToLock = _convertToShares(S, profit, Math.Rounding.Down);

          ...

                    // Mint the protocol fees to the recipient.
@=>                 _mint(S, protocolFeesRecipient, protocolFeeShares);
          ...

        // Update the new total assets value.
@=>     S.totalAssets = newTotalAssets;
        S.lastReport = uint96(block.timestamp);

        // Emit event with info
        emit Reported(
            profit,
            loss,
            protocolFees, // Protocol fees
            totalFees - protocolFees // Performance Fees
        );
}
```

## Impact

**Likelihood: Medium**\
For this to happen, there must be some WETH directly held by the strategy. Although WETH is not meant to be directly held by the strategy, as it would be immediately sold during the claimAndSwap, it is possible to donate some WETH directly to this contract. This would then be locked and erroneously included in the accounting during the harvest call.

\
**Impact: Medium**\
New fee shares will be minted for assets not held by the strategy, which will dilute the price of the share. Also, totalAssets won't represent the real assets held by the strategy.

## Tools Used

Manual review.

## Recommendations

When calculating totalAssets during the harvest and report call, two options are available:

1. Swap all WETH held by the strategy to alETH during the call.
2. Exclude WETH from the totalAssets calculation and implement a separate method that will swap WETH directly held by the contract.

The first option is more gas-consuming since it requires a swap to be executed on every report from the keeper. Therefore, the second approach is recommended. This new method is not expected to be called frequently, only when there are some locked underlying tokens.

```diff
function _harvestAndReport()
        internal
        override
        returns (uint256 _totalAssets)
    {
      ...
-     uint256 underlyingBalance = underlying.balanceOf(address(this));
-     _totalAssets = unexchanged + asset.balanceOf(address(this)) + underlyingBalance;
+     _totalAssets = unexchanged + asset.balanceOf(address(this));
    }
```

```diff
+ function swapUnderlying(
+        uint256 _minOut, 
+        uint256 _routeNumber
+    ) external onlyKeepers {
+        uint256 balBefore = asset.balanceOf(address(this));
+        router.exchange(
+            routes[_routeNumber],
+            swapParams[_routeNumber],
+            underlying.balanceOf(address(this)),
+            _minOut,
+            pools[_routeNumber],
+            address(this)
+        );        
+        uint256 balAfter = asset.balanceOf(address(this));
+        require((balAfter - balBefore) >= _minOut, "Slippage too high");
+        transmuter.deposit(asset.balanceOf(address(this)), address(this));
+    }
```


# Low Risk Findings

## <a id='L-01'></a>L-01. Hanging approvals on old router contracts can lead to the loss of funds if routers are compromised.            



## Summary

The `StrategyOp` and `StrategyArb` contracts enable the owner to change the router contract by calling the `setRouter` function.

```Solidity
   function setRouter(address _router) external onlyManagement {
        router = _router;
@=>     underlying.safeApprove(router, type(uint256).max); 
   }
```

The new router will be granted maximum approval for future swaps, but the old router will retain its maximum approval, either through a previous call to `setRouter` or during initialization if it was the first router set. If an external router is compromised, the owner can switch to a new router. However, because the old router will still have approval, there is a risk of losing underlying token funds if some tokens remain in the contract.

```Solidity
    /**
     * @dev Initializes the strategy with the router address & approves WETH to be swapped via router
    */
    function _initStrategy() internal {
        router = 0xAAA87963EFeB6f7E0a2711F397663105Acb1805e;
@=>     underlying.safeApprove(address(router), type(uint256).max);
        
    }
```

## Impact

This will be categorized as low impact because:

1. The router must be exploitable for max approval, which is unlikely for leading industry protocols today, although it's not as uncommon as in the case of Merlin DEX: [Merlin DEX Rekt](https://rekt.news/merlin-dex-rekt/).
2. Even if the router is compromised, the strategy is not expected to directly hold underlying tokens, as they should be swapped for alETH tokens. However, any underlying tokens held, such as through donations, would be at risk.

## Tools Used

Manual review.

## Recommendations

Remove previous maximum approvals for old routers when setting a new router inside `StrategyOp` and `StrategyArb`.

```diff
function setRouter(address _router) external onlyManagement {
+       underlying.safeApprove(router, 0);
        router = _router;
        underlying.safeApprove(router, type(uint256).max);
}
```



