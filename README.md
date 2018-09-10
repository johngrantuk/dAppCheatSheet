# dAppCheatSheet
My cheat sheet for dApp development

## web3.js

[web3.js 0.20.x](https://github.com/ethereum/wiki/wiki/JavaScript-API)

### Big Numbers, Wei, Eth, etc

Web3 uses the BigNumber library, number values with always be BigNumber objects as JavaScript is not able to handle big numbers correctly.

It's recommended to always keep your balance/values in wei and only transform it to other units when presenting to the user.

Useful web3 functions are: web3.toBigNumber, web3.toWei, web3.fromWei, etc.

BigNumber functions can be found [here](https://github.com/MikeMcl/bignumber.js/) and include: toNumber, toString, isEqualTo, minus, plus, etc

https://github.com/ethereum/wiki/wiki/JavaScript-API#a-note-on-big-numbers-in-web3js

### call vs sendTransaction

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

## Deploying Contract That Relies On Another Contract

Deploys the ItemUpgradeable contract, on success it then deploys the Parent contract. Then registerItem function on deployed Parent is called with ItemUpgradeable deployed address as parameter. (This was used as part of an upgradeable pattern)

[Truffle docs explanation](https://truffleframework.com/docs/getting_started/migrations#deployer).

Example Truffle migration file:

```
  var itemUpgradeableInstance;


  deployer.deploy(ItemUpgradeable).then(function (instance) {
    // ItemUpgradeable contract has been deployed now
    itemUpgradeableInstance = instance;
    return deployer.deploy(Parent);
  }).then(function(instance){
    // Parent contract has been deployed - instance

    // Calling registerItem on Parent instance
    return instance.registerItem.sendTransaction(1, itemUpgradeableInstance.address);
  }).then(function (){
    console.log('ItemUpgradeable set-up as first version with storage.')
    return
  })
```

## [Other Truffle Tips](https://medium.com/coinmonks/using-truffle-framework-in-an-advanced-way-7e32c11c97a9)

## MetaMask

Checking for active account - i.e. if a user changes account using the externsion the application should know. Couple of posts of interest are [here](https://ethereum.stackexchange.com/questions/17491/better-pattern-to-detect-web3-default-account-when-using-metamask/17523) and [here](https://ethereum.stackexchange.com/questions/42768/how-can-i-detect-change-in-account-in-metamask). In my code I used:

```
In load contracts function:

    setInterval(() => this.checkMetaMask(), 100);

Which is periodically calling this function:

  checkMetaMask() {
    // Checks for active MetaMask account info.
    if (this.state.web3.eth.accounts[0] !== this.state.account) {
      this.setState({
        account: this.state.web3.eth.accounts[0]
      })
      this.loadItems();
    }
  }

```

## Deploying To Rinkeby Using Infura & Environment Variables

Details from [Truffle tutorial](https://truffleframework.com/tutorials/using-infura-custom-provider).

First register with [Infura](https://infura.io/signup).

Install Truffles HDWalletProvider:

```npm install truffle-hdwallet-provider```

Set up environment variables for account mnemonic and Infura access token (see below for configuring .env variables).

Configure truffle.js to use rinkeby info:

```
require('babel-register')({
  ignore: /node_modules\/(?!zeppelin-solidity)/
});
require('babel-polyfill');
var HDWalletProvider = require('truffle-hdwallet-provider');
require('dotenv').load();
var mnemonic = process.env.MNEMONIC;

module.exports = {
  networks:{
    development:{
      host: "127.0.0.1",
      port: "8545",
      network_id: "*"
    },
    rinkeby: {
      provider: function() {
        return new HDWalletProvider(mnemonic, "https://rinkeby.infura.io/" + process.env.INFURA);
      },
      network_id: 4,
      gasPrice: 20000000000, // 20 GWEI
      gas: 3716887 // gas limit, set any number you want
    }
  }
};
```

Deploy to the Ropsten network:

```truffle migrate --network rinkeby```

If you want to verify that your contract was deployed successfully, you can check this on the [Rinkeby](https://rinkeby.etherscan.io/) section of Etherscan. In the search field, type in the transaction ID for your contract.

## Node Environment Variables

From [Twilio Blog.](
https://www.twilio.com/blog/2017/08/working-with-environment-variables-in-node-js.html) Using Environment variables in project to avoid saving sensitive info like Infura API keys to GitHub.

Install npm module called dotenv:

```npm install dotenv —save```

Save variable in .env file (and add .env to git ignore):

```
FOO=thisIsSensitive
```

Then use following in file where variable is required:
```
require('dotenv').load();
process.env.FOO
```

Example truffle.js:
```
require('dotenv').load();
var mnemonic = process.env.MNEMONIC;

provider: function() {
  return new HDWalletProvider(mnemonic, "https://rinkeby.infura.io/" + process.env.INFURA);
}
```

## IPFS

IPFS has two different libraries:

+ JS-IPFS, which implements the IPFS protocol in javascript.
+ JS-IPFS-API, which makes calls to an IPFS node.

### IPFS & Infura

Using the IPFS Infura node means that you don't have to run a local daemon.

+ Sign up to Infura - https://infura.io/signup
+ Use [JS-IPFS-API](https://github.com/ipfs/js-ipfs-api)
+ Install using: npm install --save ipfs-api

My IPFS helper library:

```
const IPFS = require('ipfs-api');
const ipfs = new IPFS({ host: 'ipfs.infura.io', port: 5001, protocol: 'https' });

exports.uploadInfo = async (info) => {
  console.log('IpfsUploadingInfo...');
  const data = Buffer.from(JSON.stringify(info));
  const result = await ipfs.files.add(data);
  console.log('Info Hash: ' + result[0].hash);
  return result[0].hash;
}

exports.uploadPic = async (data) => {
  console.log('IpfsUploadingPic...');
  const buf = Buffer.from(data);                                     // Convert file data into buffer
  const result = await ipfs.add(buf);
  console.log('Pic Hash: ' + result[0].hash);
  return result[0].hash;
}

exports.getItemSpecification = async (hash) => {
  const buf = await ipfs.files.cat(`/ipfs/${hash}`);
  let spec;
  try {
    spec = JSON.parse(buf.toString());
  } catch (e) {
    throw new Error(`Could not get item specification for hash ${hash}`);
  }
  return spec;
}
```
