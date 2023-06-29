# Issue H-1: Vaults can't handle the incentives distributed by the products 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/43 

## Found by 
mstpr-brainbot, nobody2018, rvierdiiev
## Summary
Vaults are eligible for product incentive rewards but lack the necessary functions to claim and distribute these rewards to users within the vault. 
## Vulnerability Detail
In the current scenario, if a product has an incentiviser, all depositors are eligible for these incentives. As vaults also make deposits into products, they are also qualified to claim rewards from the incentiviser. However, the vaults lack the functionality to claim these rewards from the incentiviser, and there isn't any existing code to distribute these rewards to the users within the vault. Even when a vault does not participate in the incentives, its presence still dilutes the share of rewards for other product users in the reward pool.
## Impact
Although the rewards can be claimed for anyone, vault has no functionality to handle the reward tokens. This will create an economical damage to reward depositor (program owner) and the users in the product hence, I'll call it as high.
## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L167
## Tool used

Manual Review

## Recommendation
Implement functionality allowing vault depositors to claim product-related rewards if they're eligible. If vaults are excluded from such rewards, their deposited balance should not factor into the incentiviser's distribution calculations, thereby increasing other product depositors' reward shares.



## Discussion

**arjun-io**

This is intended behavior. Vaults aren't part of the core protocol and we opted to not build in incentivizer reward handling in the current vault iterations (although that can be added in a future implementation of the Vaults)

# Issue H-2: BalancedVault.sol: loss of funds + global settlement flywheel / user settlement flywheels getting out of sync 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/45 

## Found by 
0xGoodess, cergyk, roguereddwarf
## Summary
When an epoch has become "stale", the `BalancedVault` will treat any new deposits and redemptions in this epoch as "pending".
This means they won't get processed by the global settlement flywheel in the next epoch but one epoch later than that.

Due to the fact that anyone can push a pending deposit or redemption of a user further ahead by making an arbitrarily small deposit in the "intermediate epoch" (i.e. the epoch between when the user creates the pending deposit / redemption and the epoch when it is scheduled to be processed by the global settlement flywheel), the user can experience a DOS.

Worse than that, by pushing the pending deposit / pending redemption further ahead, the global settlement flywheel and the user settlement flywheel get out of sync.

Also users can experience a loss of funds.

## Vulnerability Detail
Let's assume we are currently in epoch `1` and it is `stale`.
User1 calls the `deposit` function and we set `_pendingEpochs[user1] = context.epoch + 1 = 2`
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L160-L164

We move one epoch ahead and are in epoch `2` now.
By looking at the global settlement flywheel we see that `_deposit = _pendingDeposit` and `_redemption = _pendingRedemption` which means that the deposit we made will be processed in the global settlement flywheel in the next epoch (epoch `3`).
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L378-L403

By looking into the user settlement flywheel we see that the `if (accountContext.epoch > _pendingEpochs[account])` condition to process the pending deposit is not fulfilled yet (`2 !> 2`). It will be fulfilled in epoch `3`.
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L413-L422

So far so good. The global settlement flywheel and the user settlement flywheel are in sync and will process the pending deposit in epoch `3`.

Now here's the issue.
A malicious user2 or user1 unknowingly (depending on the specific scenario) calls `deposit` for user1 again in the current epoch `2` once it has become `stale` (it's possible to deposit an arbitrarily small amount).
By doing so we set `_pendingEpochs[user1] = context.epoch + 1 = 3`, thereby pushing the processing of the deposit in the user settlement flywheel one epoch ahead.

It's important to understand that the initial deposit will still be processed in epoch `3` in the global settlement flywheel, it's just being pushed ahead in the user settlement flywheel.

Thereby the global settlement flywheel and user settlement flywheel are out of sync now.

The total supply will be increased by an amount calculated based on the epoch `3` context:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L379

The user balance will be increased by an amount calculated based on the epoch `4` context:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L407

An example for a loss of funds that can occur as a result of this issue is when the PnL from epoch `3` to epoch `4` is positive. Thereby the user1 will get less shares than he is entitled to.

Similarly it is possible to push pending redemptions ahead, thereby the `_totalUnclaimed` amount would be increased by an amount that is different from the amount that `_unclaimed[account]` is increased by.

Coming back to the case with the pending deposit, I wrote a test that you can add to `BalancedVaultMulti.test.ts`:
```javascript
it('pending deposit pushed by 1 epoch causing shares difference', async () => {
      const smallDeposit = utils.parseEther('1000')
      const smallestDeposit = utils.parseEther('0.000001')

      await updateOracleEth() // epoch now stale
      // make a pending deposit
      await vault.connect(user).deposit(smallDeposit, user.address)
      await updateOracleBtc()
      await vault.sync()

      await updateOracleEth() // epoch now stale
      /* 
      user2 deposits for user1, thereby pushing the pending deposit ahead and causing the 
      global settlement flywheel and user settlement flywheel to get out of sync
      */
      await vault.connect(user2).deposit(smallestDeposit, user.address)
      await updateOracleBtc()
      await vault.sync()

      await updateOracle()
      // pending deposit for user1 is now processed in the user settlement flywheel
      await vault.syncAccount(user.address)

      const totalSupply = await vault.totalSupply()
      const balanceUser1 = await vault.balanceOf(user.address)
      const balanceUser2 = await vault.balanceOf(user2.address)

      /*
      totalSupply is bigger than the amount of shares of both users together
      this is because user1 loses out on some shares that he is entitled to
      -> loss of funds
      */
      console.log(totalSupply);
      console.log(balanceUser1.add(balanceUser2));

})
```

The impact that is generated by having one pending deposit that is off by one epoch is small.
However over time this would evolve into a chaotic situation, where the state of the Vault is significantly corrupted.

## Impact
The biggest impact comes from the global settlement flywheel and user settlement flywheel getting out of sync.
As shown above, this can lead to a direct loss of funds for the user (e.g. the amount of shares he gets for a deposit are calculated with the wrong context).

Apart from the direct impact for a single user, there is a subtler impact which can be more severe in the long term.
Important invariants are violated:
* Sum of user balances is equal to the total supply
* Sum of unclaimed user assets is equal to total unclaimed assets

Thereby the impact is not limited to a single user but affects the calculations for all users.

Less important but still noteworthy is that users that deposit into the Vault are partially exposed to PnL in the underlying products. The Vault does not employ a fully delta-neutral strategy. Therefore by experiencing a larger delay until the pending deposit / redemption is processed, users incur the risk of negative PnL.

## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L156-L175

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L184-L205

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L375-L424

## Tool used
Manual Review

## Recommendation
My recommendation is to implement a queue for pending deposits / pending redemptions of a user.
Pending deposits / redemptions can then be processed independently (without new pending deposits / redemptions affecting when existing ones are processed).

Possibly there is a simpler solution which might involve restricting the ability to make deposits to the user himself and only allowing one pending deposit / redemption to exist at a time.

The solution to implement depends on how flexible the sponsor wants the deposit / redemption functionality to be.



## Discussion

**KenzoAgada**

Also note duplicate issue #74 which mentions how a user can redeem more assets than he's entitled to.

# Issue H-3: BalancedVault.sol: Rebalancing logic can get stuck, leading to a loss of funds if a user wants to resolve it 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/66 

## Found by 
mstpr-brainbot, roguereddwarf
## Summary
Each action that is executed in the Vault (`deposit`, `redeem`, `claim`, `sync`) calls the `_rebalance` function. This function first rebalances the collateral and then the position:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L431-L434

First collateral is withdrawn, then deposited (both if needed):
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L440-L465

Then the position is reduced and increased (both if needed)
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L499-L519

The problem is that rebalancing can revert which leaves the user that wants to execute an operation in the Vault no other choice other than sending additional assets to the Vault (note that calling the `deposit` function might not be possible because rebalancing reverts). The user needs to send the funds directly to the Vault, without calling the deposit function. Thereby this would be a loss of funds for the user because the additional funds would be shared across all users.

## Vulnerability Detail
Assume e.g. that the amount of collateral that the Vault has deposited into a Product is a little bit over the minimum collateral amount.

When the Product's makers are settled at a loss, the rebalancing logic in the Vault then tries to withdraw all collateral from the Product because it is below the minimum amount.

However this causes a revert because the position has not been closed yet and zero collateral cannot maintain the position.

## Impact
The Vault can get stuck and to resolve the situation, assets need to be sent to the Vault which are then essentially lost because they are shared among all users.

## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L431-L434

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L440-L465

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L499-L519

## Tool used
Manual Review

## Recommendation
It should not be possible that the rebalancing fails because it is required for all actions that can be executed in the Vault.
However solving the issue is not as simple as changing the order of actions in the call to `_rebalance` to this:
1. reduce position
2. reduce collateral
3. increase collateral
4. increase position

That's because there is delayed settlement in the Product and a change in position size is not immediately reflected.
Therefore a solution needs to be adopted that first reduces the position and then waits for settlement until steps 2-4 are executed.



## Discussion

**KenzoAgada**

You can look at the duplicates referenced above to see additional perspectives and scenarios of this issue.

# Issue M-1: ChainlinkAggregator: binary search for roundId does not work correctly and Oracle can even end up temporarily DOSed 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/4 

## Found by 
roguereddwarf
## Summary
When a phase switchover occurs, it can be necessary that phases need to be searched for a `roundId` with a timestamp as close as possible but bigger than `targetTimestamp`.

Finding the `roundId` with the closest possible timestamp is necessary according to the sponsor to minimize the delay of position changes:

![2023-05-25_13-55](https://github.com/roguereddwarf/images/assets/118631472/0eb0a93b-1a5e-41b2-91c4-884a51aed432)

The binary search algorithm is not able to find this best `roundId` which thereby causes unintended position changes.

Also it can occur that the `ChainlinkAggregator` library is unable to find a valid `roundId` at all (as opposed to only not finding the "best").

This would cause the Oracle to be temporarily DOSed until there are more valid rounds.

## Vulnerability Detail
Let's look at the binary search algorithm:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-oracle/contracts/types/ChainlinkAggregator.sol#L123-L156

The part that we are particularly interested in is:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-oracle/contracts/types/ChainlinkAggregator.sol#L139-L149

Let's say in a phase there's only one valid round (`roundId=1`) and the timestamp for this round is greater than `targetTimestamp`

We would expect the `roundId` that the binary search finds to be `roundId=1`.

The binary search loop is executed with `minRoundId=1` and `maxRoundId=1001`.

All the above conditions can easily occur in reality, they represent the basic scenario under which this algorithm executes.

`minRoundId` and `maxRoundId` change like this in the iterations of the loop:

```text
minRoundId=1
maxRoundId=1001

-> 

minRoundId=1
maxRoundId=501

-> 

minRoundId=1
maxRoundId=251

-> 

minRoundId=1
maxRoundId=126

-> 

minRoundId=1
maxRoundId=63

-> 

minRoundId=1
maxRoundId=32

-> 

minRoundId=1
maxRoundId=16

-> 

minRoundId=1
maxRoundId=8

-> 

minRoundId=1
maxRoundId=4

-> 

minRoundId=1
maxRoundId=2

Now the loop terminates because
minRoundId + 1 !< maxRoundId

```

Since we assumed that `roundId=2` is invalid, the function returns `0` (`maxTimestamp=type(uint256).max`):

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-oracle/contracts/types/ChainlinkAggregator.sol#L153-L155

In the case that `latestRound.roundId` is equal to the `roundId=1`  (i.e. same phase and same round id which could not be found) there would be no other valid rounds that the `ChainlinkAggregator` can find which causes a temporary DOS.

## Impact
As explained above this would result in sub-optimal and unintended position changes in the best case.
In the worst-case the Oracle can be temporarily DOSed, unable to find a valid `roundId`.

This means that users cannot interact with the perennial protocol because the Oracle cannot be synced.
So they cannot close losing trades which is a loss of funds.

The DOS can occur since the while loop searching the phases does not terminate:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-oracle/contracts/types/ChainlinkAggregator.sol#L88-L91

## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-oracle/contracts/types/ChainlinkAggregator.sol#L123-L149

## Tool used
Manual Review

## Recommendation
I recommend to add a check if `minRoundId` is a valid solution for the binary search.
If it is, `minRoundId` should be used to return the result instead of `maxRoundId`:

```diff
         // If the found timestamp is not greater than target timestamp or no max was found, then the desired round does
         // not exist in this phase
-        if (maxTimestamp <= targetTimestamp || maxTimestamp == type(uint256).max) return 0;
+        if ((minTimestamp <= targetTimestamp || minTimestamp == type(uint256).max) && (maxTimestamp <= targetTimestamp || maxTimestamp == type(uint256).max)) return 0;
 
+        if (minTimestamp > targetTimestamp) {
+            return _aggregatorRoundIdToProxyRoundId(phaseId, uint80(minRoundId));
+        }
         return _aggregatorRoundIdToProxyRoundId(phaseId, uint80(maxRoundId));
     }
```

After applying the changes, the binary search only returns `0` if both `minRoundId` and `maxRoundId` are not a valid result.

If this line is passed we know that either of both is valid and we can use `minRoundId` if it is the better result.

# Issue M-2: Payoff definitions that can cross zero price are not supported 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/40 

## Found by 
roguereddwarf
## Summary
The price that is returned by an Oracle can be transformed by a "Payoff Provider" to transform the Oracle price and implement different payoff functions (e.g. 3x Leveraged ETH, ETH*ETH).

According to the documentation, any payoff function is supported:
https://docs.perennial.finance/mechanism/payoff
![2023-06-04_09-44](https://github.com/roguereddwarf/images/assets/118631472/1251a80f-107d-47b3-bac7-c6e8e12485b2)

The problem is that this is not true.

## Vulnerability Detail
Payoff functions that transform the Oracle price such that the transformed price can cross from negative to positive or from positive to negative are not supported.

This is because there is a location in the code, where there is division by `currentPrice`:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L487

Thus there is a risk of division-by-zero if the price was allowed to cross the 0 value.

## Impact
In contrast to what is specified in the Docs, not all payoff functions are supported which can lead to a situation where it causes division-by-zero and thereby DOS.

## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L487

## Tool used
Manual Review

## Recommendation
Establish which payoff functions are supported and make it clear in the docs or find a meaningful way to handle the above case.
If `currentPrice==0`, a meaningful value could be to set `currentPrice=1`



## Discussion

**arjun-io**

We'll update the docs to clarify some payoffs which aren't supported

# Issue M-3: If long and short products has different maker fees, vault rebalance can be spammed to eat vaults balance 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/41 

## Found by 
mstpr-brainbot
## Summary
In a market scenario where long and short products carry different position fees, continuous rebalancing could potentially deplete the vault's funds. This situation arises when closing positions incurs a maker fee, which gradually eats away at the collateral balance. The rebalancing cycle continues until the fee becomes insignificant or the collateral balance is exhausted.
## Vulnerability Detail
If a market's long and short products have different position fees, continuous rebalancing could be an issue. A significant position fee difference might deplete all funds after repeated rebalancing.

Consider a scenario with one market, where the long product has a 0% maker fee (position fee), and the short product a 10% maker fee. A vault holding 100 positions divided across these markets, with 50 short and 50 long positions, and collateral balances of 50-50 each, is presented.

Now, suppose the vault and the product are entirely in sync (meaning they're of the same version). If someone deposits 100 assets into the vault, it would distribute that deposit, adding 50 to both product's collateral balances, bringing them to 100 each. The vault then increases its position to 100 to maintain the 1x leverage. The long market would match this since it has no maker fee, but the short market would lose 5 from its collateral due to the fee charged by the openMake operation. As a result, the market now has 95 in the short market and 100 in the long market collateral. This imbalance triggers a rebalancing operation via sync().

After the second call to sync(), the collateral balances would be 97.5 in both markets, and the markets would reduce their positions to maintain the 1x leverage. The long market would reduce its position to 97.5, but when the short market does the same, the closeMake operation's fee charge reduces the short market collateral by 0.25, resulting in final balances of 97.5 in the long market and 97.25 in the short market. This imbalance again allows a call to settle...

As seen if the difference between the products are high the rebalance will start eating the vaults collateral balance by paying redundant maker fees.

The above scenario assumes only one market with two products. If there are two markets, the situation could be more troublesome due to the additional fee charges and potentially higher losses.
## Impact
Since the vaults funds can be drained significantly if certain cases are met (high makerFee set by market owner and big collateral balance differences between markets or directional exposure on vault causing the collateral balance differences higher than usual so more funds to rebalance hence more fee to pay). I'll label it as high. 
## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L137-L149

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L426-L533

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L341-L355

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/collateral/Collateral.sol#L147-L153
## Tool used

Manual Review

## Recommendation
Consider accounting the positionFee if there are any on any of the markets or assert that both of the markets has the same positionFee. 



## Discussion

**arjun-io**

This is a good call - currently vault's don't support handling maker fees in a clean way. We probably won't fix this in the current version of the vaults but it's something we'd likely address in the future as maker fees might become necessary in certain markets

# Issue M-4: BalancedVault.sol: Early depositor can manipulate exchange rate and steal funds 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/46 

## Found by 
Phantasmagoria, roguereddwarf
## Summary
The first depositor can mint a very small number of shares, then donate assets to the Vault.
Thereby he manipulates the exchange rate and later depositors lose funds due to rounding down in the number of shares they receive.

The currently deployed Vaults already hold funds and will merely be upgraded to V2. However as Perennial expands there will surely be the need for more Vaults which enables this issue to occur.

## Vulnerability Detail
You can add the following test to `BalancedVaultMulti.test.ts`.
Make sure to have the `dsu` variable available in the test since by default this variable is not exposed to the tests.

The test is self-explanatory and contains the necessary comments:

```javascript
it('exchange rate manipulation', async () => {
      const smallDeposit = utils.parseEther('1')
      const smallestDeposit = utils.parseEther('0.000000000000000001')

      // make a deposit with the attacker. Deposit 1 Wei to mint 1 Wei of shares
      await vault.connect(user).deposit(smallestDeposit, user.address)
      await updateOracle();
      await vault.sync()

      console.log(await vault.totalSupply());

      // donating assets to Vault
      await dsu.connect(user).transfer(vault.address, utils.parseEther('1'))

      console.log(await vault.totalAssets());

      // make a deposit with the victim. Due to rounding the victim will end up with 0 shares
      await updateOracle();
      await vault.sync()
      await vault.connect(user2).deposit(smallDeposit, user2.address)
      await updateOracle();
      await vault.sync()

      console.log(await vault.totalAssets());
      console.log(await vault.totalSupply());
      // the amount of shares the victim receives is rounded down to 0
      console.log(await vault.balanceOf(user2.address));

      /*
      at this point there are 2000000000000000001 Wei of assets in the Vault and only 1 Wei of shares
      which is owned by the attacker.
      This means the attacker has stolen all funds from the victim.
      */
    })
```

## Impact
The attacker can steal funds from later depositors.

## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L775-L778

## Tool used
Manual Review

## Recommendation
This issue can be mitigated by requiring a minimum deposit of assets.
Thereby the attacker cannot manipulate the exchange rate to be so low as to enable this attack.



## Discussion

**arjun-io**

The inflation attack has been reported and paid out on Immunefi (happy to provide proof here if needed) - we have added a comment describing this attack here: https://github.com/equilibria-xyz/perennial-mono/pull/194


# Issue M-5: BalancedVault.sol: V2 upgrade does not implement V1 functions which means users can lose access to their funds 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/56 

## Found by 
roguereddwarf
## Summary
The current `BalancedVault` is meant as an upgrade for the current single market implementation of the `BalancedVault`.

We can see this in the initializer (initializing version 2):
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L113

The issue is that shares in the V1 Vault are transferrable
https://github.com/equilibria-xyz/perennial-mono/blob/5ba9c9fc594cc476e9370bd68eb5ac7fd04bfc10/packages/perennial-vaults/contracts/BalancedVault.sol#L215-L233

```solidity
    function transfer(address to, UFixed18 amount) external returns (bool) {
        _settle(msg.sender);
        _transfer(msg.sender, to, amount);
        return true;
    }

    /**
     * @notice Moves `amount` shares from `from to `to`
     * @param from Address to send shares from
     * @param to Address to send shares to
     * @param amount Amount of shares to send
     * @return bool true if the transfer was successful, otherwise reverts
     */
    function transferFrom(address from, address to, UFixed18 amount) external returns (bool) {
        _settle(from);
        _consumeAllowance(from, msg.sender, amount);
        _transfer(from, to, amount);
        return true;
    }
```

However in the V2 implementation the `transfer` and `transferFrom` functions have been removed.

## Vulnerability Detail
The vulnerability can occur because based on V1 the `shares` might be managed by a contract that does not itself have the ability to redeem them.

However with the upgrade to V2, the assets corresponding to the shares get lost because the contract would not be able to transfer the shares somewhere else where they can be redeemed.

## Impact
Loss of funds because shares could not get redeemed.

Note that this scenario does not rely on a user error.
The shares might be owned by a smart contract and be subject to the logic of the contract. Thereby it's not possible that the smart contract just transfers them somewhere where they can be redeemed before the V2 upgrade.

## Code Snippet
V2 `BalancedVault`:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L30

V1 `BalancedVault`:
https://github.com/equilibria-xyz/perennial-mono/blob/master/packages/perennial-vaults/contracts/BalancedVault.sol

## Tool used
Manual Review

## Recommendation
Implement the `transfer` and `transferFrom` functions in the V2 `BalancedVault` such as to not break compatibility.



## Discussion

**KenzoAgada**

Essentially report is valid, but practically there are only 2 vaults, and looking at Arbscan there doesn't seem to be any contracts holding vault tokens. (But that's just an eye test, not thorough check).
But even if so, we also can not guarantee it will stay this way - by the time the vaults will be upgraded, there might be contracts holding tokens. I think medium severity is appropriate.

**arjun-io**

We intentionally removed this functionality. We verified that no contracts currently hold tokens, and we'll verify that again before deploying this upgrade.

# Issue M-6: BalancedVault.sol: claim can be impossible due to unsigned integer underflow 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/57 

## Found by 
roguereddwarf
## Summary
When a user redeems his shares, he then needs to call the `claim` function to actually claim his assets.

This will downstream call the `_rebalancePosition` function with the `claimAmount` as an argument.

Under some circumstances the `_rebalancePosition` function can revert due to unsigned integer underflow, making it impossible to claim assets.

This condition is not permanent but can only be solved by sending additional assets to the Vault causing a loss of funds.

## Vulnerability Detail
Assume the following situation:
```text
_assets() = 5e18
_totalUnclaimed = 10e18

unclaimed of user1 = 5e18
```

When user1 calls `claim`, pro-rata claiming applies and the actual `claimAmount` is equal to `5e18*10e18/10e18=5e18`.

Now let's look at the vulnerable line:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L474

`_totalAssetsAtEpoch` would be equal to `0`:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L684-L690

Thereby the line in `_rebalancePosition` reverts due to unsigned integer underflow (`0 - claimAmount`),

You can add the following test to `BalancedVaultMulti.test.ts` which is a modified version of the existing `gracefully unwinds upon insolvency` test:
```javascript
it('reverts upon pro rata claim', async () => {
        // 1. Deposit initial amount into the vault
        await vault.connect(user).deposit(utils.parseEther('50000'), user.address)
        await vault.connect(user2).deposit(utils.parseEther('50000'), user2.address)
        await updateOracle()
        await vault.sync()

        // 2. Redeem most of the amount, but leave it unclaimed
        
        await vault.connect(user).redeem(utils.parseEther('40000'), user.address)
        await vault.connect(user2).redeem(utils.parseEther('20000'), user2.address)
        await updateOracle()
        await vault.sync()

        // 3. An oracle update makes the long position liquidatable, initiate take close
        await updateOracle(utils.parseEther('20000'))
        await long.connect(user).settleAccount(vault.address)
        await short.connect(user).settleAccount(vault.address)
        await long.connect(perennialUser).closeTake(utils.parseEther('700'))
        await collateral.connect(liquidator).liquidate(vault.address, long.address)

        // // 4. Settle the vault to recover and rebalance
        await updateOracle() // let take settle at high price
        await updateOracle(utils.parseEther('1500')) // return to normal price to let vault rebalance
        await vault.sync()
        await updateOracle()
        await vault.sync()

        // 6. Claim should be pro-rated
        await vault.claim(user2.address)
})
```


## Impact
The user is at least temporarily unable to claim his assets and the situation needs to be solved by sending additional assets to the Vault (of which the user most likely does not get the majority back with the claim) -> loss of funds

## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L211-L228

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L472-L492

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L684-L690

## Tool used
Manual Review

## Recommendation
In the `_rebalancePosition` function it must be ensured that the following line does not execute when `claimAmount > _totalAssetsAtEpoch(context)`:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L474

# Issue M-7: Malicious trader can bypass utilization buffer 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/75 

## Found by 
ast3ros, cergyk
## Summary
A malicious trader can bypass the Utilisation buffer, and push utilisation to 1 on any product.

## Vulnerability Detail
Perenial products use a utilization buffer to prevent taker side to DOS maker by taking up all the liquidity:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L547-L556

Makers would not be able to withdraw if utilization would reach 100% because of `takerInvariant` which is enforced during `closeMake`:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L535-L544

A malicious trader can bypass the utilisation buffer by first opening a maker position, open taker position and close previous maker position.
The only constraint is that she has to use different accounts to open the maker positions and the taker positions, since perennial doesn't allow to have maker and taker positions on a product simultaneously.

So here's a concrete example:

Let's say the state of product is `900 USD` on the maker side, `800 USD` on the taker side, which respects the 10% utilization buffer.

### Example
>Using account 1, Alice opens up a maker position for `100 USD`, bringing maker total to `1000 USD`.

>Using account 2, Alice can open a taker position for `100 USD`, bringing taker to `900 USD`, which respects the 10% utilization buffer still.

>Now using account 1 again, Alice can close her `100 USD` maker position and withdraw collateral, clearing account 1 on perennial completely.

>This brings the utilization to `100%`, since taker = maker = `900 USD`. 

>This is allowed, since only `takerInvariant` is checked when closing a maker position, which enforces that utilization ratio is lower than or equal to `100%`.

## Impact
Any trader can bring the utilization up to 100%, and use that to DoS withdrawals from Products and Balanced vaults for an indefinite amount of time.
This is especially critical for vaults, since when any product related to any market is fully utilized, all redeems from the balanced vault are blocked.

## Code Snippet

## Tool used

Manual Review

## Recommendation
If a safety buffer needs to be enforced for the utilisation of products, it needs to be enforced on closing make positions as well.



## Discussion

**arjun-io**

This is a good callout and a valid issue but since the _impact_ existed before this utilization buffer feature was added we disagree with the severity. The utilization buffer is not intended to totally fix the 100% utilization issue against malicious users, but instead provide a buffer in the normal operating case.

**KenzoAgada**

In addition to the sponsor's comment, also worth noting that a malicious user will not gain anything from doing this.
Indeed seems like medium severity is more appropriate. Downgrading to medium.

# Issue M-8: A trader close to liquidation risks being liquidated by trying to reduce her position 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/76 

## Found by 
cergyk
## Summary
A trader close to liquidation risks being liquidated by trying to reduce her position

## Vulnerability Detail
We can see in the implementation of closeTake/closeMake that the `maintenanceInvariant` is not enforced:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L246-L256

and the takerFee is deducted immediately from traders collateral, whereas the position reduction only takes effect at next settlement:
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L258-L273

This means that a legitimate user close to liquidation, who may want to reduce leverage by closing position will end up immediately liquidatable.

Example:
>Maintenance factor is 2%
>Liquidation fee is 5%
>Taker fee is 0.1%
>Funding fee is 0%

Alice has a 50x leverage `taker` position opened as authorized per maintenance invariant:

Alice collateral: 100 DSU
Alice taker position: notional of 5000 DSU

Alice tries to reduce her position to a more reasonable 40x, calls `closeTake(1000)`. 
A 1 DSU taker fee is deducted immediately from her collateral. Since the reduction of the position only takes effect on next oracle version, her position becomes immediately liquidatable. 
A liquidator closes it entirely, making Alice lose an additional 4 DSU `takerFee` on next settlement and 250 DSU of liquidationFee.

## Impact
A user trying to avoid liquidation is liquidated unjustly

## Code Snippet

## Tool used
Manual Review

## Recommendation
Either:
- Do not deduce takerFee from collateral immediately, but at next settlement, as any other fees
or
- Enforce `maintenanceInvariant` on closing of positions, to make transaction revert (except when closing is liquidation already)

# Issue M-9: User would liquidate his account to sidestep `takerInvariant` modifier 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/77 

## Found by 
Emmanuel, branch\_indigo
## Summary
A single user could open a massive maker position, using the maximum leverage possible(and possibly reach the maker limit), and when a lot of takers open take positions, maker would liquidate his position, effectively bypassing the taker invariant and losing nothing apart from position fees.
This would cause takers to be charged extremely high funding fees(at the maxRate), and takers that are not actively monitoring their positions will be greatly affected. 

## Vulnerability Detail
In the closeMakeFor function, there is a modifier called `takerInvariant`.
```solidity
function closeMakeFor(
        address account,
        UFixed18 amount
    )
        public
        nonReentrant
        notPaused
        onlyAccountOrMultiInvoker(account)
        settleForAccount(account)
        takerInvariant
        closeInvariant(account)
        liquidationInvariant(account)
    {
        _closeMake(account, amount);
    }
```
This modifier prevents makers from closing their positions if it would make the global maker open positions to fall below the global taker open positions.
A malicious maker can easily sidestep this by liquidating his own account.
Liquidating an account pays the liquidator a fee from the account's collateral, and then forcefully closes all open maker and taker positions for that account.
```solidity
function closeAll(address account) external onlyCollateral notClosed settleForAccount(account) {
        AccountPosition storage accountPosition = _positions[account];
        Position memory p = accountPosition.position.next(_positions[account].pre);

        // Close all positions
        _closeMake(account, p.maker);
        _closeTake(account, p.taker);

        // Mark liquidation to lock position
        accountPosition.liquidation = true; 
    }
```
This would make the open maker positions to drop significantly below the open taker position, and greatly increase the funding fee and utilization ratio.

### ATTACK SCENARIO
- A new Product(ETH-Long) is launched on arbitrum with the following configurations:
    - 20x max leverage(5% maintenance)
    - makerFee = 0
    - takerFee = 0.015
    - liquidationFee = 20%
    - minRate = 4%
    - maxRate = 120%
    - targetRate = 12%
    - targetUtilization = 80%
    - makerLimit = 4000 Eth
    - ETH price = 1750 USD
    - Coll Token = USDC
    - max liquidity(USD) = 4000*1750 = $7,000,000
- Whale initially supplies 350k USDC of collateral(~200ETH), and opens a maker position of 3000ETH($5.25mn), at 15x leverage.
- After 2 weeks of activity, global open maker position goes up to 3429ETH($6mn), and because fundingFee is low, people are incentivized to open taker positions, so global open taker position gets to 2743ETH($4.8mn) at 80% utilization. Now, rate of fundingFee is 12%
- Now, Whale should only be able to close up to 686ETH($1.2mn) of his maker position using the `closeMakeFor` function because of the `takerInvariant` modifier.
- Whale decides to withdraw 87.5k USDC(~50ETH), bringing his total collateral to 262.5k USDC, and his leverage to 20x(which is the max leverage)
- If price of ETH temporarily goes up to 1755 USD, totalMaintenance=3000 * 1755 * 5% = $263250. Because his totalCollateral is 262500 USDC(which is less than totalMaintenance), his account becomes liquidatable.
- Whale liquidates his account, he receives liquidationFee*totalMaintenance = 20% * 263250 = 52650USDC, and his maker position of 3000ETH gets closed. Now, he can withdraw his remaining collateral(262500-52650=209850)USDC because he has no open positions.
- Global taker position is now 2743ETH($4.8mn), and global maker position is 429ETH($750k)
- Whale has succeeded in bypassing the takerInvaraiant modifier, which was to prevent him from closing his maker position if it would make global maker position less than global taker position.

Consequently,
- Funding fees would now be very high(120%), so the currently open taker positions will be greatly penalized, and takers who are not actively monitoring their position could lose a lot.
- Whale would want to gain from the high funding fees, so he would open a maker position that would still keep the global maker position less than the global taker position(e.g. collateral of 232750USDC at 15x leverage, open position = ~2000ETH($3.5mn)) so that taker positions will keep getting charged at the funding fee maxRate.


## Impact
User will close his maker position when he shouldn't be allowed to, and it would cause open taker positions to be greatly impacted. And those who are not actively monitoring their open taker positions will suffer loss due to high funding fees.

## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/collateral/Collateral.sol#L123-L132
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L362-L372
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L334
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L535-L544


## Tool used

Manual Review

## Recommendation
Consider implementing any of these:
- Protocol should receive a share of liquidation fee: This would disincentivize users from wanting to liquidate their own accounts, and they would want to keep their positions healthy and over-collateralized
- Let there be a maker limit on each account: In addition to the global maker limit, there should be maker limit for each account which may be capped at 5% of global maker limit. This would decentralize liquidity provisioning.




## Discussion

**arjun-io**

This is working as intended. The market dynamics (namely funding rate curve) should incentivize more makers to come into the market in this case, although it is possible for a malicious maker to temporarily increase funding rates in situations where there is not other LPs (or vaults).

# Issue M-10: Liquidating an account may cause collateral balance of the account to go below `minCollateral` 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/103 

## Found by 
Emmanuel
## Summary
There is a minimum collateral that is set in the Controller contract. This disallows users from depositing collateral that would make collateral balance less than minimum Collateral, and prevents withdraws that would make the collateral balance less than minimum collateral.
This issue explains how an account can cause his collateral balance to go below the minCollateral and even open positions with it, hence defeating the purpose of minCollateral

## Vulnerability Detail
Here is how the protocol calculates the liquidation fee that would be deducted from the account.
```solidity
UFixed18 liquidationFee = controller().liquidationFee();
UFixed18 collateralForFee = UFixed18Lib.max(totalMaintenance, controller().minCollateral());
UFixed18 fee = UFixed18Lib.min(totalCollateral, collateralForFee.mul(liquidationFee)); 
_products[product].debitAccount(account, fee);
```
The `fee` is the minimum of `totalCollateral` and `collateralForFee.mul(liquidationFee)` which means that if fee is not `totalCollateral`, there is a possibility that the fee could be large enough to make `totalCollateral` fall below `minCollateral`

Consider this scenario:
- There is ETH-long product on Ethereum Mainnet with following configurations:
    - minimum collateral of 100 USDC
    - liquidation fee = 20%
    - maintenance=0.05, max leverage=20
    - ETH price=$1750
- UserA deposit collateral of 100 USDC(equal to min collateral)
- UserA opens maker position of 1.143ETH at 20x leverage
- Price of ETH rises to $1800 so account immediately becomes liquidatable because account used max leverage
- UserA liquidates his account:
    - collateralForFee=max((1800 * 1.143 * 0.05),100)=$102.87
    - fee=min(100,(102.87 * 20%))=$20.574
- liquidation fee that would be deducted from UserA's collateral balance is $20.574
- UserA's account balance=100-20.574=$79.43 which is less than minCollateral of $100
- UserA can now open new positions using his collateral balance that is less than minCollateral

One of the purposes of minCollateral is to allow for gas costs that would cover liquidation.
- UserA can calculate a collateral balance(liquidationBalance) that would be less than gas costs on ethereum to dissuade liquidators from liquidating his account even when his collateral balance is well below the required maintenance.
- UserA opens a position using max leverage to make his account easily liquidatable
- UserA liquidates his account to make his collateral balance=liquidationBalance
- UserA would open positions using liquidationBalance as collateral, and be rest assured that liquidators won't want to liquidate his account.

NOTE: There is no strict mechanism to liquidate account when it falls below minCollateral

## Impact
A user can cause his collateral balance to be any value, even below minCollateral. With this power, he can make his collateral balance such that there would be no incentive to liquidate it.
Now, even when massive changes in Product's token price causes his collateral balance to be far less than the maintenance required, User's account may not be liquidated as there are no incentives.

## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/collateral/Collateral.sol#L126-L132


## Tool used

Manual Review

## Recommendation
Create a mapping called "unused collateral"
When liquidating an account, if deducting liquidation fee causes account's balance to be less than minCollateral, increment the account's "unused collateral" balance by the account's balance, and change the `OptimisticLedger` balance of that account to 0.
The "unused collateral" balances should not be used in any internal accounting, and should be claimable by the user at anytime.

# Issue M-11: Position fees will cause collateral balance of an account to go below `minCollateral` 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/123 

## Found by 
Emmanuel
## Summary
There is a minimum collateral that is set in the Controller contract. This disallows users from depositing collateral that would make collateral balance less than minimum Collateral, and prevents withdraws that would make the collateral balance less than minimum collateral.
This issue explains how an account can cause his collateral balance to go below the minCollateral and even open positions with it, hence defeating the purpose of minCollateral

## Vulnerability Detail
Opening a position debits a takerFee or makerFee, depending on the position opened, from the collateral balance, and then checks if the collateral will be enough after the next settlement.
The problem is that the functions do not check if the collateral balance, after deducting the fees, is more than minCollateral. This would allow the following:
- User deposits collateral that is just a little above minCollateral
- User opens and closes positions, so position fees makes the balance less than minCollateral
- User repeats above step till his balance reaches the point when liquidators pay a higher gas fee than the liquidation fee he would receive
- User opens a risky position, but liquidators will be dissuaded from liquidating it

One of the purposes of minCollateral is to allow for gas costs that would cover liquidation.
- User can calculate a collateral balance(liquidationBalance) that would be less than gas costs on ethereum to dissuade liquidators from liquidating his account even when his collateral balance is well below the required maintenance.
- User would then open and close a position that would make his collateral balance liquidationBalance after fees are deducted
- User would open positions using liquidationBalance as collateral, and be rest assured that liquidators won't want to liquidate his account.

NOTE: There is no strict mechanism to liquidate account when it falls below minCollateral
## Impact
A user can cause his collateral balance to be any value, even below minCollateral. With this power, he can make his collateral balance such that there would be no incentive to liquidate it.
Now, even when massive changes in Product's token price causes his collateral balance to be far less than the maintenance required, User's account may not be liquidated as there are no incentives for doing so.

## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L225
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L266
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L307
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L349

## Tool used

Manual Review

## Recommendation
- Allow liquidation of an account whose collateral balance has fallen below minCollateral.(This would take care of instances when fundingFees causes the collateral balance to fall below minCollateral)
- Alternatively, Create a modifier in openMake and openTake founctions that ensures that after debiting the open+close position fees, the collateral balance does not fall below minCollateral



## Discussion

**arjun-io**

This is a good callout - the minCollateral value should be set to give ample buffer to allow for safe liquidations even if the balance falls below this but it is a good case to note.

# Issue M-12: Accounts will not be liquidated when they are meant to. 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/132 

## Found by 
0xGoodess, Emmanuel, mstpr-brainbot, simon135
## Summary
In the case that the totalMaintenance*liquidationFee is higher than the account's totalCollateral, liquidators are paid the totalCollateral. I think one of the reasons for this is to avoid the case where liquidating an account would attempt to debit fees that is greater than the collateral balance
The problem is that, the value of totalCollateral used as fee is slightly higher value than the current collateral balance, which means that in such cases, attempts to liquidate the account would revert due to underflow errors.

## Vulnerability Detail
Here is the `liquidate` function:
```solidity
function liquidate(
        address account,
        IProduct product
    ) external nonReentrant notPaused isProduct(product) settleForAccount(account, product) {
        if (product.isLiquidating(account)) revert CollateralAccountLiquidatingError(account);

        UFixed18 totalMaintenance = product.maintenance(account); maintenance?
        UFixed18 totalCollateral = collateral(account, product); 

        if (!totalMaintenance.gt(totalCollateral))
            revert CollateralCantLiquidate(totalMaintenance, totalCollateral);

        product.closeAll(account);

        // claim fee
        UFixed18 liquidationFee = controller().liquidationFee();
      
        UFixed18 collateralForFee = UFixed18Lib.max(totalMaintenance, controller().minCollateral()); 
        UFixed18 fee = UFixed18Lib.min(totalCollateral, collateralForFee.mul(liquidationFee)); 

        _products[product].debitAccount(account, fee); 
        token.push(msg.sender, fee);

        emit Liquidation(account, product, msg.sender, fee);
    }
```
`fee=min(totalCollateral,collateralForFee*liquidationFee)`
But the PROBLEM is, the value of totalCollateral is fetched before calling `product.closeAll`, and `product.closeAll` debits the closePosition fee from the collateral balance. So there is an attempt to debit `totalCollateral`, when the current collateral balance of the account is `totalCollateral`-closePositionFees
This allows the following:
- There is an ETH-long market with following configs:
    - maintenance=5%
    - minCollateral=100USDC
    - liquidationFee=20%
    - ETH price=$1000
- User uses 500USDC to open $10000(10ETH) position
- Price of ETH spikes up to $6000
- Required maintenance= 60000*5%=$3000 which is higher than account's collateral balance(500USDC), therefore account should be liquidated
- A watcher attempts to liquidate the account which does the following:
    - totalCollateral=500USDC
    - `product.closeAll` closes the position and debits a makerFee of 10USDC
    - current collateral balance=490USDC
    - collateralForFee=totalMaintenance=$3000
    - fee=min(500,3000*20%)=500
    - `_products[product].debitAccount(account,fee)` attempts to subtract 500 from 490 which would revert due to underflow
    - account does not get liquidated
- Now, User is not liquidated even when he is using 500USD to control a $60000 position at 120x leverage(whereas, maxLeverage=20x)

NOTE: This would happen when the market token's price increases by (1/liquidationFee)x. In the above example, price of ETH increased by 6x (from 1000USD to 6000USD) which is greater than 5(1/20%)

## Impact
A User's position will not be liquidated even when his collateral balance falls WELL below the required maintenance. I believe this is of HIGH impact because this scenario is very likely to happen, and when it does, the protocol will be greatly affected because a lot of users will be trading abnormally high leveraged positions without getting liquidated.

## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/collateral/Collateral.sol#L118
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/collateral/Collateral.sol#L123
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/collateral/Collateral.sol#L129-L132

## Tool used

Manual Review

## Recommendation
`totalCollateral` that would be paid to liquidator should be refetched after `product.closeAll` is called to get the current collateral balance after closePositionFees have been debited.



## Discussion

**KenzoAgada**

See the `Recommendation` section above for the root of the issue.

# Issue M-13: Unintended Vault Operation Due to Product Settling and Oracle Version Skips 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/152 

## Found by 
mstpr-brainbot
## Summary
Vault operation can behave unexpectedly when products settle and skip versions in their oracles. This problem arises when the vault assumes a non-existent version of a product, leading to miscalculations in the _assetsAtEpoch() function. This inaccuracy can affect the distribution of shares among depositors, redeemers, and claimers, disrupting the balance of the market.
## Vulnerability Detail
When a product progresses to a new version of its oracle, it first settles at the current version plus one, then jumps to the latest version. For instance, if a product's latest version is 20 and the current version is 23, the product will first settle at version 21 and then jump from 21 to 23, bypassing version 22. This process can lead to unintended behavior in the vault, particularly if the vault's deposited epoch points to the bypassed version, which can result in a potential underflow and subsequent rounding to zero.

Consider this scenario: A vault is operating with two markets, each having two products. Let's assume the vault has equal positions and collateral in both markets, with all assets priced at $1 for simplicity. Also, assume that Market0 contains product A and product B, while Market1 contains product C and product D.

The latest version of product A is 20, while the vault's latest epoch is 2, which corresponds to version 20 for product A. Similarly, the latest version of product C is 10, with the vault's latest epoch at 2, corresponding to version 10 for product A.

Assume there are no positions created in product C, D, and also no deposits made to the vault, meaning the vault has not opened any positions on these products either. The oracle version for product C, D gets updated twice consecutively. Since there are no open positions in product C, D, their products will progress to version 12 directly, bypassing version 11.

This circumstance renders the epoch stale, which will lead to miscalculations in deposits, claims, and redemptions. During the _assetsAtEpoch() function, the vault incorrectly assumes that there is a version 11 of product C, D. However, these products have skipped version 11 and advanced to 12. Since the latest version of product C, D is greater than the version the vault holds, the vault presumes that version 11 exists. However, as version 11 doesn't exist, the _accumulatedAtEpoch will be the negative of whatever was deposited. This leads the vault to incorrectly assume a large loss in that market. Consequently, depositors can gain more shares, while redeemers and claimers may receive fewer shares.

## Impact
If the above scenario happens, vault users can incur extreme losses on claim and redeem operations and depositors can earn uneven shares.
## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/product/Product.sol#L84-L130
settle

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial/contracts/interfaces/types/PrePosition.sol#L133-L136
if no open positions in preposition settle version is current version

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L376

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L823-L843
checks for +1 version, if latestVersion > currentVersion, it assumes vault lost all the funds in that market.

https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L797-L805

## Tool used

Manual Review

## Recommendation
Don't check +1 version, check the current version recorded for that epoch in vault and the latest version of the product, cover all the versions 



## Discussion

**kbrizzle**

It is guaranteed that all applicable versions have a corresponding checkpoint in the underlying markets at version `n` because product.settle() is called on every market ([here](https://github.com/equilibria-xyz/perennial-mono/blob/dev/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L586-L589) from [here](https://github.com/equilibria-xyz/perennial-mono/blob/dev/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L377)) when an [epoch snapshot](https://github.com/equilibria-xyz/perennial-mono/blob/dev/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L394) takes place.

Further, when there is a deposit or withdrawal `_updateMakerPosition` will create a new position in the underlying product [here](https://github.com/equilibria-xyz/perennial-mono/blob/master/packages/perennial-vaults/contracts/BalancedVault.sol#L440-L443), which will create a checkpoint for `n + 1`.

**KenzoAgada**

Suspected as much but wanted to make sure... Closing as invalid.

**KenzoAgada**

After internal discussion: the issue is valid, but the watson didn't fully identify the exact complex edge case where it will materialize. Therefore downgrading to medium severity.

# Issue M-14: Users can be forced to claim assets at bad rate in some cases 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/174 

## Found by 
mstpr-brainbot, rvierdiiev, tvdung94
## Summary
In balanced vault contract, anyone can call claim assets for other accounts; malicious users could abuse this function to force other users to receive less assets than they expect.
## Vulnerability Detail
When claiming asset, pro rate, in which users will receive less assets than expect, will be applied when total collateral is less than total unclaimed amount.  So after redeeming shares and converting it to assets, users might not want to claim assets right away in this scenario, for they will receive less token amount. However, other users can force them to claim via claim() function because there is no restrict on this function to claim for other accounts.
## Impact
Users will be forced to receive less assets in some scenarios.
## Code Snippet
https://github.com/sherlock-audit/2023-05-perennial/blob/main/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L211-L228
## Tool used

Manual Review

## Recommendation
Only account owner (msg.sender == account) can claim asset for their account.



## Discussion

**KenzoAgada**

While the general claim pro-rata mechanism has already been reported in the Veriside audit (issue 002) and acknowledged by the protocol, this submission correctly adds that a user can pro-rata claim *other user's* assets, even if they preferred to avoid realizing losses and stay in the vault until collateral recovers. This will make users lose funds, and increase the malicious user's own amount of rewards, for when the collateral recovers.

**arjun-io**

This is a great find. We will think about whether or not we want to fix this as the unpermissioned claim is currently used by the protocol for better UX with the MultiInvoker

# Issue M-15: `BalancedVault` doesn't consider potential break in one of the markets 

Source: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/232 

## Found by 
BLACK-PANDA-REACH, Bauchibred, mstpr-brainbot
## Summary
In case of critical failure of any of the underlying markets, making it permanently impossible to close position and withdraw collateral all funds deposited to balanced Vault will be lost, including funds deposited to other markets.

## Vulnerability Detail

As Markets and Vaults on Perennial are intented to be created in a permissionless manner and integrate with external price feeds, it cannot be ruled out that any Market will enter a state of catastrophic failure at a point in the future (i.e. oracle used stops functioning and Market admin keys are compromised, so it cannot be changed), resulting in permanent inability to process closing positions and withdrawing collateral.

`BalancedVault` does not consider this case, exposing all funds deposited to a multi-market Vault to an increased risk, as it is not implementing a possibility for users to withdraw deposited funds through a partial emergency withdrawal from other markets, even at a price of losing the claim to locked funds in case it becomes available in the future. This risk is not mentioned in the documentation.

### Proof of Concept
Consider a Vault with 2 markets: ETH/USD and ARB/USD.

1. Alice deposits to Vault, her funds are split between 2 markets
2. ARB/USD market undergoes a fatal failure resulting in `maxAmount` returned from `_maxRedeemAtEpoch` to be 0
3. Alice cannot start withdrawal process as this line in `redeem` reverts:
```solidity
    if (shares.gt(_maxRedeemAtEpoch(context, accountContext, account))) revert BalancedVaultRedemptionLimitExceeded();
``` 

## Impact
Users funds are exposed to increased risk compared to depositing to each market individually and in case of failure of any of the markets all funds are lost. User has no possibility to consciously cut losses and withdraw funds from Markets other than the failed one.

## Code Snippet
`_maxRedeemAtEpoch`
https://github.com/sherlock-audit/2023-05-perennial/blob/0f73469508a4cd3d90b382eac2112f012a5a9852/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L649-L677

check in `redeem`
https://github.com/sherlock-audit/2023-05-perennial/blob/0f73469508a4cd3d90b382eac2112f012a5a9852/perennial-mono/packages/perennial-vaults/contracts/balanced/BalancedVault.sol#L188

## Tool used

Manual Review

## Recommendation
Implement a partial/emergency withdrawal or acknowledge the risk clearly in Vault's documentation.



## Discussion

**KenzoAgada**

Please note that the duplicate issues referenced above mention paused markets and oracle fails as additional scenarios where markets can break and therefore break the vault. Issue #20 contains recommendations on how to handle stuck epochs and oracles.

