# dAppCheatSheet
My cheat sheet for dApp development

## web3.js

[web3.js 0.20.x](https://github.com/ethereum/wiki/wiki/JavaScript-API)

## call vs sendTransaction

#### [web3.eth.call(callObject [, defaultBlock] [, callback])](https://github.com/ethereum/wiki/wiki/JavaScript-API#web3ethcall)
It only runs on the local node, and won't create a transaction on the blockchain.

Use with view and pure functions (i.e. function that don’t change state of Blockchain & don’t cost gas)

Returns a String - The functions return value.

Example:
```
let instance = await ItemContract.deployed();
let itemInfo = await instance.getItem.call(1, {from: accounts[0]});
let itemOwner = itemInfo[0];
let itemBounty = itemInfo[1].toNumber();
```

#### [web3.eth.sendTransaction(transactionObject [, callback])](https://github.com/ethereum/wiki/wiki/JavaScript-API#web3ethsendtransaction)
Sends a transaction to the network. Can change data on the blockchain.

Note: sending a transaction will require the user to pay gas, and will pop up their Metamask to prompt them to sign a transaction.

Returns String - The 32 Bytes transaction hash as HEX string. (See below for how the Tx Hash can be used)

```
let bounty_amount = web3.toWei(1, 'ether');
let instance = await ItemContract.deployed();
let itemInfo = 'testInfo';

let hash = await instance.makeItem.sendTransaction(itemInfo, {value: bounty_amount, from: accounts[0]});
```
## Test Tips

#### Testing Value Transfer

Can use transaction hash returned from sendTransaction to calculate gas used whan making the transaction.

```
    it("should make item with correct bounty", async () => {
      let bounty_amount = web3.toWei(1, 'ether');
      let instance = await ItemContract.deployed();
      let itemInfo = 'testInfo';

      let account_one_starting_balance = await web3.eth.getBalance(accounts[0]);

      let hash = await instance.makeItem.sendTransaction(itemInfo, {value: bounty_amount, from: accounts[0]});            // Creates Item - sends bounty & costs gas to do work

      const tx = await web3.eth.getTransaction(hash);
      const receipt = await web3.eth.getTransactionReceipt(hash);                                                           // Calculates used Gas for Create Item
      const gasCost = tx.gasPrice.mul(receipt.gasUsed);

      let account_one_ending_balance = await web3.eth.getBalance(accounts[0]);
      let account_one_ending_balance_check = account_one_starting_balance.minus(gasCost);
      account_one_ending_balance_check = account_one_ending_balance_check.minus(bounty_amount);                           // This is all deductions

      assert.equal(account_one_ending_balance.toNumber(), account_one_ending_balance_check.toNumber(), "Bounty amount for make item wasn't correctly taken from the creator");
    });
```

#### Testing For A Throw

Use a helper function to test for a throw:

```
// This is a helper function to test for throws.
const expectThrow = async (promise) => {
      try {
        await promise;
      } catch (error) {
        const invalidJump = error.message.search('invalid JUMP') >= 0;
        const outOfGas = error.message.search('out of gas') >= 0;
        const revert = error.message.search('revert') >= 0;
        assert(
          invalidJump || outOfGas || revert,
          "Expected throw, got '" + error + "' instead",
        );
        return;
      }
      assert.fail(0, 1, 'Expected throw not received');
}

it("get non-existing item should throw.", async () => {
  let instance = await ItemContract.deployed();
  expectThrow(instance.getItem(99));
})
```
