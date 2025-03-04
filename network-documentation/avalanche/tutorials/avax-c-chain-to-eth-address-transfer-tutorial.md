# Introduction

In this tutorial, we are going to learn how to programatically transfer native AVAX tokens from the Avalanche C-Chain to a Metamask wallet.

# Prerequisites

* [Create an Avalanche wallet](https://wallet.avax.network/create)
* [Fund your Avalanche wallet with the FUJI Faucet](https://docs.avax.network/build/tutorials/platform/fuji-workflow#get-a-drip-from-the-fuji-faucet)
* [Transfer FUJI Avax tokens from X-chain to C-chain](https://docs.avax.network/build/tutorials/platform/transfer-avax-between-x-chain-and-c-chain)
* Having an integrated development environment, such as [Visual Studio Code](https://code.visualstudio.com/download)

# Requirements

* [NodeJS](https://nodejs.org/en)
* [Ethers](https://docs.ethers.io/v5/) and [web3](https://web3js.readthedocs.io/en/v1.5.2/), which you can install with `npm install ethers web3`
* Install [Metamask extension](https://metamask.io/download.html) in your browser.
* [Configure your Metamask to add Avalanche FUJI testnet](https://docs.avax.network/build/tutorials/smart-contracts/deploy-a-smart-contract-on-avalanche-using-remix-and-metamask#step-1-setting-up-metamask)  

# Transfer AVAX 

We are going to start by creating a new file called `AVAX_C_chain_to_ETH_address.js` in the root directory of your project. Once you create a .js file under the specified name, we will type in the following blocks of code in. 

First, we need to import Ethers and Web3 libraries mentioned under requirements in order to interact with the Avalanche C chain.

```javascript
const Web3 = require("web3")
const { ethers } = require('ethers')
```

The mnemonic key from your AVAX wallet needs to go between the quotation marks below. This is later used to extract the C-Chain wallet address.

```javascript
let mnemonic = "";
```

This code below is pointing to the Fuji network. Alternatively, we can point it to the Datahub AVAX node.

```javascript
const web3 = new Web3(new Web3.providers.HttpProvider("https://api.avax-test.network/ext/bc/C/rpc"))
```

Alternatively, the DataHub version would look as follows:

```javascript
const web3 = new Web3(new Web3.providers.HttpProvider(`https://avalanche--fuji--rpc.datahub.figment.io/apikey/${process.env.DATAHUB_AVALANCHE_API_KEY}/ext/bc/C/rpc`))
```

The DataHub version example above refers to the API key stored in the `.env` file. Refer to [Getting started with dotenv and .env files](https://docs.figment.io/network-documentation/extra-guides/dotenv-and-.env) for help. Sign up for a DataHub account to acquire an API key and store it in the `.env` file. 

The private key is needed to execute a transfer later on. With the mnemonic phrase provided earlier, we can obtain the private key to your AVAX wallet in the ETH address format with the colde below.

```javascript
const wallet = new ethers.Wallet.fromMnemonic(mnemonic);
AVAX_privatekey = wallet.privateKey;
```

The web3 package is designed to interact with an Ethereum blockchain and Ethereum smart contracts. The web3 module is capable of acting upon the AVAX implementation of the EVM. So, when interacting with the C chain, the web3 module is used. Here, `web3.eth.getBalance` fetches the AVAX balance. `result` in `web3.utils.fromWei` below is the balance of the account in the units of wei \(the minimum unit of Ether is called wei and 1 Ether is 10^18 wei\). `fromWei` is a method in web3.utils, converting a number from one unit to another. So, `web3.utils.fromWei(result, "ether")` in the code below converts the balance from wei to ether. Since we are viewing the AVAX token balance, we will print AVAX at the end.

```javascript
async function main(){
    web3.eth.getBalance(wallet.address, function(err, result) {    
        if (err) {
          console.log(err)
        } else {
          console.log(web3.utils.fromWei(result, "ether") + " AVAX")
        }
    })
```
`console.log(web3.utils.fromWei(result, "ether") + " AVAX")` is what outputs the current balance on the C-chain of the sender's wallet. 

The block below is to view the \# of transactions associated with the wallet address \(not necessary for the purpose of AVAX transfer from C chain to an ETH address\) but could be useful when you later want to transfer an ERC20 token.

```javascript
    web3.eth.getTransactionCount(wallet.address)       
    .then(console.log);                                               
    const createTransaction = await web3.eth.accounts.signTransaction(           
        {
           gas: 21000,
```

`web3.eth.getTransactionCount(wallet.address).then(console.log)` is the portion of the script that outputs the number of transactions that have happened in the account so far. 

Moving on, copy and paste your Metamask wallet address (or any ETH format address) beween the quotation marks below

```javascript
           to:"",
```

Below, we need to input how many tokens are to be transferred out of your AVAX wallet off the C chain. Currently the amount is arbitrarily set to 0.15 AVAX tokens as an example \(0.15 tokens in the line below\).

```javascript
           value: web3.utils.toWei('0.15', 'ether'),     
        },
        AVAX_privatekey                                 
     );
     const createReceipt = await web3.eth.sendSignedTransaction(
        createTransaction.rawTransaction
     );
     console.log(
        `Transaction successful with hash: ${createReceipt.transactionHash}`
     );
  };
main().catch((err) => {
    console.log("We have encountered an error!")
    console.error(err)
})
```

At this point, the transfer should have completed. \`Transaction successful with hash: ${createReceipt.transactionHash}\` outputs the transaction hash to the terminal. This is your transaction record.

At this point, you have gone through the entire script. 

The finished script should look as follows:

```javascript
const Web3 = require("web3")
const { ethers } = require('ethers')
let mnemonic = "";
const web3 = new Web3(new Web3.providers.HttpProvider("https://api.avax-test.network/ext/bc/C/rpc"))
const wallet = new ethers.Wallet.fromMnemonic(mnemonic);
AVAX_privatekey = wallet.privateKey;

async function main(){
    web3.eth.getBalance(wallet.address, function(err, result) {    
        if (err) {
          console.log(err)
        } else {
          console.log(web3.utils.fromWei(result, "ether") + " AVAX")
        }
    })
    web3.eth.getTransactionCount(wallet.address)       
    .then(console.log);                                               
    const createTransaction = await web3.eth.accounts.signTransaction(           
        {
           gas: 21000,
           to:"",
           value: web3.utils.toWei('0.15', 'ether'),     
        },
        AVAX_privatekey                                 
     );
     const createReceipt = await web3.eth.sendSignedTransaction(
        createTransaction.rawTransaction
     );
     console.log(
        `Transaction successful with hash: ${createReceipt.transactionHash}`
     );
  };
main().catch((err) => {
    console.log("We have encountered an error!")
    console.error(err)
})
```

To run the script `AVAX_C_chain_to_ETH_address.js`, type `node AVAX_C_chain_to_ETH_address.js` and run it in your terminal (`node` before the file name is the way to invoke the NodeJS runtime environment).

# Conclusion

This tutorial has taught you how to transfer AVAX native tokens from the C chain to an ETH wallet. Also, this has shown how Avalanche blockchain C chain is compatible with the popular web3.js library. This is a powerful aspect of the Avalanche blockchain, as it allows Ethereum developers to easily port their work over to Avalanche.

Try transferring your Fuji AVAX tokens by running this script and see if it worked.

When everything was run as expected, the output in the terminal should look similar to what we have below.

![example](https://i.imgur.com/yr6nkto.png)

# Troubleshooting

## Transaction Failure

* Check if you have sufficient balance in your Avalanche wallet. 

Not having sufficient balance would result in the following error in your terminal. 

![example](https://i.imgur.com/Jh2a6Yl.png)  

Make sure that your balance on the C-chain is sufficient as shown below

![example](https://i.imgur.com/d2QFBU0.png)

# About the author

This tutorial was created by [Seongwoo Oh](https://github.com/blackwidoq). He is a student and an Avalanche novice.

