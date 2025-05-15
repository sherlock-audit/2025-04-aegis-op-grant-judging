# Issue H-1: Insolvency as `YUSD` will depeg overtime as the redemption fees are disbursed with no collaterals backing them. 

Source: https://github.com/sherlock-audit/2025-04-aegis-op-grant-judging/issues/1 

## Found by 
0xNForcer, 0xapple, 0xaxaxa, 0xeix, 0xpiken, 0xrex, Asher, Fade, FalseGenius, Kvar, QuillAudits\_Team, Ryonen, aslanbek, asui, bigbear1229, farman1094, gesha17, gkrastenov, harry, hildingr, iamandreiski, irorochadere, itsgreg, mahdifa, mgnfy.view, molaratai, moray5554, onthehunt, phoenixv110, s0x0mtee, shealtielanz, silver\_eth, t0x1c, turvec, vinica\_boy, w33kEd, x0x0




## Summary
On the call to `approveRedeemRequest()` the collateral amount which was used initially to get the `yusdAmount` in the `mint()` function, is calculated via the `_calculateRedeemMinCollateralAmount()`.

Here's a run down, please take a good gander.
```solidity

File: AegisMinting.sol

 function approveRedeemRequest(string calldata requestId, uint256 amount) external nonReentrant onlyRole(FUNDS_MANAGER_ROLE) whenRedeemUnpaused {
    RedeemRequest storage request = _redeemRequests[keccak256(abi.encode(requestId))];
    if (request.timestamp == 0 || request.status != RedeemRequestStatus.PENDING) {
      revert InvalidRedeemRequest();
    }
    if (amount == 0 || amount > request.order.collateralAmount) {
      revert InvalidAmount();
    }

    uint256 collateralAmount = _calculateRedeemMinCollateralAmount(request.order.collateralAsset, amount, request.order.yusdAmount);
//code snip ... some checks.
    uint256 availableAssetFunds = _untrackedAvailableAssetBalance(request.order.collateralAsset);
    if (availableAssetFunds < collateralAmount) {
      revert NotEnoughFunds();
    }

    // Take a fee, if it's applicable
    // @audit  fee amount is taken without accounting for the collateral backing thereby leading to a depegging of the YUSD
    (uint256 burnAmount, uint256 fee) = _calculateInsuranceFundFeeFromAmount(request.order.yusdAmount, redeemFeeBP);
    if (fee > 0) {
      yusd.safeTransfer(insuranceFundAddress, fee);
    }

    request.status = RedeemRequestStatus.APPROVED;
    totalRedeemLockedYUSD -= request.order.yusdAmount;
   //@audit transfers the entire collateral amount to the user.
    IERC20(request.order.collateralAsset).safeTransfer(request.order.userWallet, collateralAmount);
    yusd.burn(burnAmount);

    emit ApproveRedeemRequest(requestId, _msgSender(), request.order.userWallet, request.order.collateralAsset, collateralAmount, burnAmount, fee);
  }

```
The issue is simply, it sends the entire collateral amount backing the given `yusdAmount` to the user, the takes a fee out of the `yusdAmount` and burns the rest, but given fees left aren't backed by any collateral the fee are worthless but would still be redeemable for any of the collateral assets in the contract thereby causing negative yield for the protocol.
As fees accumulate, as seen in the code the fees can even rise to 50% it faster the higher the redemption fees are and are accumulated.


## Severity  Justification.
- While this might at first  not seem like a big deal anyone understanding the dynamics of the protocol would see that over time, the stables backing the `YUSD` will be depeleted causing the price of `YUSD` to crash make the contracts insolvent.


## Mitigation

The collateral amount to be sent back to the user should be the equivalence in value of the the `yusd` to be burnt on the call to  `approveRedeemRequest()`
 The code snippet should be changed as follows.
 ```diff
 function approveRedeemRequest(string calldata requestId, uint256 amount) external nonReentrant onlyRole(FUNDS_MANAGER_ROLE) whenRedeemUnpaused {
    RedeemRequest storage request = _redeemRequests[keccak256(abi.encode(requestId))];
    if (request.timestamp == 0 || request.status != RedeemRequestStatus.PENDING) {
      revert InvalidRedeemRequest();
    }
    if (amount == 0 || amount > request.order.collateralAmount) {
      revert InvalidAmount();
    }
++   // Take a fee, if it's applicable
++  (uint256 burnAmount, uint256 fee) = _calculateInsuranceFundFeeFromAmount(request.order.yusdAmount, redeemFeeBP);
++  if (fee > 0) {
++  yusd.safeTransfer(insuranceFundAddress, fee);
++    }

--  uint256 collateralAmount = _calculateRedeemMinCollateralAmount(request.order.collateralAsset, amount, request.order.yusdAmount);
++  uint256 collateralAmount = _calculateRedeemMinCollateralAmount(request.order.collateralAsset, amount, burnAmount);
    /*
     * Reject if:
     * - asset is no longer supported
     * - smallest amount is less than order minAmount
     * - order expired
     */
    if (
      !_supportedAssets.contains(request.order.collateralAsset) ||
      collateralAmount < request.order.slippageAdjustedAmount ||
      request.order.expiry < block.timestamp
    ) {
      _rejectRedeemRequest(requestId, request);
      return;
    }

    uint256 availableAssetFunds = _untrackedAvailableAssetBalance(request.order.collateralAsset);
    if (availableAssetFunds < collateralAmount) {
      revert NotEnoughFunds();
    }

--  // Take a fee, if it's applicable
--  (uint256 burnAmount, uint256 fee) = _calculateInsuranceFundFeeFromAmount(request.order.yusdAmount, redeemFeeBP);
--  if (fee > 0) {
--    yusd.safeTransfer(insuranceFundAddress, fee);
--  }

    request.status = RedeemRequestStatus.APPROVED;
    totalRedeemLockedYUSD -= request.order.yusdAmount;

    IERC20(request.order.collateralAsset).safeTransfer(request.order.userWallet, collateralAmount);
    yusd.burn(burnAmount);

    emit ApproveRedeemRequest(requestId, _msgSender(), request.order.userWallet, request.order.collateralAsset, collateralAmount, burnAmount, fee);
  }
 ```


# Issue M-1: Collateral can get stuck in the minting contract under certain conditions or be accounted as profit 

Source: https://github.com/sherlock-audit/2025-04-aegis-op-grant-judging/issues/77 

## Found by 
iamandreiski

### Summary

The predominant narrative of Aegis is that they aren't holding any funds in their minting contract. This means that all collateral which was transferred to the contract for the purpose of minting YUSD is transferred to their custodial partners for safekeeping OR is accounted for as profit which would result in YUSD being minted in exchange for it, and then said collateral transferred to their custodial partners for safekeeping.

This warrants the need for a 2-step redeem flow which first registers the redeem request, and then withdraws the needed collateral from the custodial partners after which the redeem can be accepted and the collateral transferred to the user. 
The problem with this approach is that these funds are "untracked", together with the profit.

There are multiple ways in which collateral can get "stuck" in the contract OR be mistaken for profit:
- A redeem request was rejected or cancelled;
- The redeem request acceptance reverts due to any number of reasons such as the user was placed on a blacklist and/or the order has expired;
- A `depositIncome` call mistakenly accounts untracked collateral funds as profit, resulting in the minting of non-collateralized YUSD which contributes to its de-pegging.

### Root Cause

The current design of the system doesn't account (virtually) for both the funds which are to-be-redeemed, as well as the income (i.e. profit). The problem of not accounting for the redeem funds is that they can either be mistaken for an income, or remain stuck in the contract with no way to return them to the custodial partners.

When a user requests a redeem, the request is placed and pending as part of `_redeemRequests`:

```solidity
    _redeemRequests[keccak256(abi.encode(requestId))] = RedeemRequest(RedeemRequestStatus.PENDING, order, block.timestamp);

```

Once the "untracked" funds become available in the minting contract (i.e. are withdrawn from the custodial partners), the redeem request can be accepted/approved:

```solidity

    uint256 availableAssetFunds = _untrackedAvailableAssetBalance(request.order.collateralAsset);
    if (availableAssetFunds < collateralAmount) {
      revert NotEnoughFunds();
    }


  function _untrackedAvailableAssetBalance(address _asset) internal view returns (uint256) {
    uint256 balance = IERC20(_asset).balanceOf(address(this));
    if (balance < _custodyTransferrableAssetFunds[_asset] + assetFrozenFunds[_asset]) {
      return 0;
    }


    return balance - _custodyTransferrableAssetFunds[_asset] - assetFrozenFunds[_asset];
  }
```
The problem with this approach is that redeem requests are expected to fail / get cancelled after the funds were withdrawn from the custodial partners to the minting contract.

- User could cancel their redeem requests after the funds were already withdrawn to the contract, but before the request is approved;
- An exchange ratio change has prompted the call to be rejected due to slippage;
- The user was placed on a blacklist and the call reverts;
- The order has expired;

Any of (but not limited to) the above-mentioned scenarios could result in the user not being able to claim their collateral, the request being rejected/cancelled and the funds remaining stuck in the minting contract.
Since currently, there's no way to send untracked funds to the custodial partners, they will remain stuck within the minting contract.

This is also a problem if the redeem request was a very large amount which affects the balancing strategy of Aegis, and not being able to utilize those funds as part of their delta-neutral position strategy.

A second point of this report is that if the funds stuck in the contract are "mistaken" or otherwise intentionally declared as profit, this would lead to the minting of undercollateralized YUSD.

Considering that the "extra" collateral in the minting contract can be declared as profit by minting adequate and corresponding YUSD amounts through `depositIncome`, the function assumes that the untracked collateral in-question wasn't utilized for the minting of YUSD tokens:

https://github.com/sherlock-audit/2025-04-aegis-op-grant/blob/main/aegis-contracts/contracts/AegisMinting.sol#L407-L424

And that's why it's being treated as a normal mint flow in which the user provides collateral and receives a corresponding amount of YUSD tokens in exchange for the collateral supplied. 

### Internal Pre-conditions

1. A certain collateral amount is deposited to the minting contract from the custodial partners for the purpose of approving redeem requests;
2. Said redeem requests can't be approved due to a multitude of reasons and are rejected or cancelled, leading to the already withdrawn "untracked" collateral to be stuck in the contract and unable to be sent back to the custodial counterparts. 

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

The untracked collateral amounts withdrawn to the minting contract which are unused (i.e. haven't been utilized for redeems) will remain stuck in the contract or can be declared as profit which will lead to the minting of undercollateralized YUSD.

### PoC

_No response_

### Mitigation

Include an admin controlled mechanism to be able to repurpose unused redeem funds as `_custodyTransferrableAssetFunds` or make a separate request funds flow which will track the amounts and enable redeem approvals (and if they get rejected, the funds get atomically repurposed). 

# Issue M-2: The _checkMintRedeemLimit is not use properly in mint lead to update wrong amount 

Source: https://github.com/sherlock-audit/2025-04-aegis-op-grant-judging/issues/328 

## Found by 
0xapple, Flare, MaslarovK, OpaBatyo, SafetyBytes, SamuelTroyDomi, SarveshLimaye, Weed0607, ck, covey0x07, durov, eLSeR17, farismaulana, farman1094, hildingr, leopoldflint, mahdifa, momentum, moray5554, newspacexyz, pkqs90, rudhra1749, tobi0x18, w33kEd, y4y, zhanmingjing

### Summary

There is function `AegisMinting::_checkMintRedeemLimit` which make sure the amount has not mint more than the limit of period, but that is not implemented or used properly which lead to check the price comparison with wrong amount or update the wrong mint amount


### Root Cause

This is the funciton which make sure the token is not mint more then the limit of period. Also update the mint amount every time mint called [here](https://github.com/sherlock-audit/2025-04-aegis-op-grant/blob/main/aegis-contracts/contracts/AegisMinting.sol#L785)
```solidity
`AegisMinting::_checkMintRedeemLimit`
  function _checkMintRedeemLimit(MintRedeemLimit storage limits, uint256 yusdAmount) internal {
    if (limits.periodDuration == 0 || limits.maxPeriodAmount == 0) {
      return;
    }
    uint256 currentPeriodEndTime = limits.currentPeriodStartTime + limits.periodDuration;
    if (
      (currentPeriodEndTime >= block.timestamp && limits.currentPeriodTotalAmount + yusdAmount > limits.maxPeriodAmount) ||
      (currentPeriodEndTime < block.timestamp && yusdAmount > limits.maxPeriodAmount)
    ) {
      revert LimitReached();
    }
    // Start new mint period
    if (currentPeriodEndTime <= block.timestamp) {
      limits.currentPeriodStartTime = uint32(block.timestamp);
      limits.currentPeriodTotalAmount = 0;
    }

@>    limits.currentPeriodTotalAmount += yusdAmount;
  }
```

Issue lies in how this function used in `AegisMinting::mint` 
```solidity
  function mint(
    OrderLib.Order calldata order,
    bytes calldata signature
  ) external nonReentrant onlyWhitelisted(order.userWallet) onlySupportedAsset(order.collateralAsset) {
    if (mintPaused) {
      revert MintPaused();
    }
    if (order.orderType != OrderLib.OrderType.MINT) {
      revert InvalidOrder();
    }

 @>   _checkMintRedeemLimit(mintLimit, order.yusdAmount);
```

The value we are sending here is come from the off chain api call signed data [check here](https://github.com/Aegis-im/aegis-contracts/blob/master/docs/mint-redeem.md). But the price is coming here is not the exact price which is going to mint.

There is another function `AegisMinting::_calculateMinYUSDAmount` in which we are taking price from chainlink, and creating another mint amount and using which ever is minimum between both yusdAmount (coming from offchain), chainlinkYUSDAmount (calculated now).
```solidity
  function _calculateMinYUSDAmount(address collateralAsset, uint256 collateralAmount, uint256 yusdAmount) internal view returns (uint256) {
    (uint256 chainlinkPrice, uint8 feedDecimals) = _getAssetUSDPriceChainlink(collateralAsset);
    if (chainlinkPrice == 0) {
      return yusdAmount;
    }

    uint256 chainlinkYUSDAmount = Math.mulDiv(
      collateralAmount * 10 ** (18 - IERC20Metadata(collateralAsset).decimals()),
      chainlinkPrice,
      10 ** feedDecimals
    );

    // Return smallest amount
@>    return Math.min(yusdAmount, chainlinkYUSDAmount);
  }
```

The price we are going to mint is `mintAmount` the minimum one.
```solidity
// AegisMinting::mint
    // Take a fee, if it's applicable
   @> (uint256 mintAmount, uint256 fee) = _calculateInsuranceFundFeeFromAmount(yusdAmount, mintFeeBP);
    if (fee > 0) {
      yusd.mint(insuranceFundAddress, fee);
    }

    IERC20(order.collateralAsset).safeTransferFrom(order.userWallet, address(this), order.collateralAmount);
 @>  yusd.mint(order.userWallet, mintAmount);
    _custodyTransferrableAssetFunds[order.collateralAsset] += order.collateralAmount;

    emit Mint(_msgSender(), order.collateralAsset, order.collateralAmount, mintAmount, fee);
  }
```


### Internal Pre-conditions

Chainlink must be returning price.

### External Pre-conditions

..

### Attack Path

Work on every call

### Impact

There is no surely we are always going to use the `yusdAmount` as the `chainlinkYUSDAmount` can be lower as well. But we are updating the state before even its confirming the mint amount.
```solidity
  function mint(
    OrderLib.Order calldata order,
    bytes calldata signature
  ) external nonReentrant onlyWhitelisted(order.userWallet) onlySupportedAsset(order.collateralAsset) {
    if (mintPaused) {
      revert MintPaused();
    }
    if (order.orderType != OrderLib.OrderType.MINT) {
      revert InvalidOrder();
    }

    _checkMintRedeemLimit(mintLimit, order.yusdAmount);
```
Which could lead to updating the wrong state value in `AegisMint::_checkMintRedeemLimit` not the actual mint amount which can be different.
```solidity
@>     limits.currentPeriodTotalAmount += yusdAmount;
```
So it could create discrepancy updating the higher amount currentPeriodTotalAmount when the mint amount can be lower. 
There is 2 possible issue it would lead to
1) Updated the higher then actual price
2) Comparing the right mint amount with wrong updated amount value


### PoC

I believe it's so obvoius and self explaining so not providing POC right now, In case required happy to provide it 


### Mitigation

update the value after confirm the correct mint amount

# Issue M-3: A whale adversary can grief the redeem functionality through redeem limit consumption 

Source: https://github.com/sherlock-audit/2025-04-aegis-op-grant-judging/issues/497 

## Found by 
0xDLG, 0xJoyBoy03, 0xNForcer, 0xbakeng, 0xc0ffEE, Ironsidesec, OpaBatyo, farman1094, gkrastenov, hildingr, iamandreiski, itsgreg, mahdikarimi, phoenixv110, ptsanev, vinica\_boy, xAlismx

### Summary

The failure to not account redemptions into the limit after the transaction has been approved can lead to griefing attempts by users with sizeable YUSD token balance meanwhile the redemption can still expire.

### Root Cause

In [`AegisMinting.sol:785-802`](https://github.com/sherlock-audit/2025-04-aegis-op-grant/blob/main/aegis-contracts/contracts/AegisMinting.sol#L785-L803), the _checkMintRedeemLimit function increments limits.currentPeriodTotalAmount when a user creates a redeem request, but neither the withdrawRedeemRequest function (lines 373-390) nor the _rejectRedeemRequest function (lines 702-711) decrements this counter when YUSD is returned to the user.

```solidity
function _checkMintRedeemLimit(MintRedeemLimit storage limits, uint256 yusdAmount) internal {
    // ... validation checks ...
    limits.currentPeriodTotalAmount += yusdAmount; // Incremented but never decremented on withdraw/reject
}
```

### Internal Pre-conditions

N/A

### External Pre-conditions

1. Adversary needs to have enough YUSD tokens to make redeem requests that approach redeemLimit.maxPeriodAmount.

### Attack Path

1. Adversary identifies the current redeemLimit.maxPeriodAmount and redeemLimit.periodDuration.
2. Adversary creates multiple valid Order structs for OrderType.REDEEM with different nonces and additionalData (to create unique request IDs).
3. Attacker obtains valid signatures for these orders from the trustedSigner.
4. Attacker calls requestRedeem with these orders multiple times, locking YUSD each time.
5. Attacker continues step 4 until redeemLimit.currentPeriodTotalAmount approaches redeemLimit.maxPeriodAmount.
6. The limit available for redeem is now almost exhausted for the current period.
7. Any user (including legitimate users) calling requestRedeem for more than the small remaining limit will have their transaction revert with LimitReached error.
8. After order.expiry has passed, the attacker calls withdrawRedeemRequest for each request.
9. The attacker receives back all their locked YUSD, but redeemLimit.currentPeriodTotalAmount remains unchanged.
10. The DoS persists until the current period ends and redeemLimit resets naturally.

### Impact

Legitimate users cannot execute the core functionality of requesting a redeem (requestRedeem) for the duration of the limit period under attack. This DoS prevents users from exiting their positions when desired, which could lead to financial loss if users need to redeem during specific market conditions. The attacker only temporarily locks their funds and loses nothing but gas fees in the process.

### PoC

Check Attack Path

### Mitigation

Adjust for the limit after the redemption has actually been approved to ensure whales have fully commited with redeeming.


