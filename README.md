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

Returns String - The 32 Bytes transaction hash as HEX string.

```
let bounty_amount = web3.toWei(1, 'ether');
let instance = await ItemContract.deployed();
let itemInfo = 'testInfo';

let hash = await instance.makeItem.sendTransaction(itemInfo, {value: bounty_amount, from: accounts[0]});
```
