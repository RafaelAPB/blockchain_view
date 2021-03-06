# Overview
This repo demonstrates a minimum prototype that can deploy a contract, invoke a transaction, query the state, inspect a block and dissect a transaction. 

# Prerequisite 
* Install `docker` (Version>19.03.8) and `docker-compose` (Version>1.25.4). [Link](https://www.docker.com/products/docker-desktop)
* Install `node` (Version>v13.13.0) and `npm` (Version>v6.14.4) [Link](https://nodejs.org/en/download/current/)

# Run
**NOTE**: All the following commands are executed under this README directory. 
## Install Frontend Dependency
Install dependent node modulers. 
```
npm install
```
## Spin up the network
```
PRIVATE_CONFIG=ignore docker-compose up -d
```
* The above command will launch a 4-node Quorum network. Each node is listenining to port 20000, 20001, 20002, 20003 in localhost (0.0.0.0). 
* `PRIVATE_CONFIG=ignore` disables all functionalities for txn privacy.

### Deploy the contract
* Deploy `simplestorage.go`
```
node deploy.js simplestorage.sol http://0.0.0.0:22000
```
The console shall display the following for success.
```
Successfully deployed the contract simplestorage. 
The deployment txn <txn_hash> is committed in Block <blk_num>. 
The ABI and address of the contract are written to simplestorage.json.
```

The above command will deploy the `simplestorage.sol` onto the Quorum chain. 
* Firstly, the script will invoke solidity compiler `solr` to compile the code into the EVM bytecode and ABI (Line 14-17 of `deploy.js`). ABI is similar to a functional interface. 
* Then, the scrirpt will contact Node 1 `http://0.0.0.0:22000` in Quorum and fetch for its account key. 
* THe deployment request is organized into a transaction signed by the key and sent to Node 1. Node 1 will submit the transaction for consensus. 
* During the consensus, each peer will execute the `constructor()` method in `simplestorage.sol`. The method will set `deployer` to the transaction/message sender, which the account of Node 1. 
* After the consensus, the script will receive the contract address from the txn receipt. Together with the above ABI, the address will be pesisted to a file `simplestorage.json`. This file with ABI and contract address will be loaded later for invocation or query. 

### Invoke the transaction
* Invoke the contract at Node 2 with its account key to set the value to 88
```
node invoke.js http://0.0.0.0:22001 simplestorage.json set 88
```

If successful, the console shall display sequentially in an interval of a few seconds. 
This is the duration of the consensus. 
```
Txn <txn_hash> has been submitted for consensus. 
Txn <txn_hash> has been committed in Block <blk-num>.
```
* This transaction will call `set` function with the argument `88` in `simplestorage.go`. 
* The transaction will first be sent to Node 2 at (http://0.0.0.0:22001), then get submmitted for the consensus and finally get committed by all peers.

#### With Access Control
Suppose we invoke `setbydeployer` also from Node 2 with its account key
```
node invoke.js http://0.0.0.0:22001 simplestorage.json setbydeployer 88
```
The invocation shall fail with the console output:
```
Txn <txn_hash> has been submitted for consensus. 
Fail with err msg: Transaction has been reverted by the EVM:
<txn details...>
```
It is due to the fact that this method is tagged with `onlyOwner` modifier.
The modifier restricts that the valid invocation must be sent from this contract deployer, which is Node 1's account. 
Hence, the invocation `setbydeployer` on Node 1 shall execute as normal: 
```
node invoke.js http://0.0.0.0:22000 simplestorage.json setbydeployer 88
```

### Query the state
```
node query.js http://0.0.0.0:22002 simplestorage.json get
```

If successful, the console instantly prints the previously set value, 88.  
```
The query result is 88. 
```
* The query will call `get` function in `simplestorage.go`.
* Such query will not undergo the consensus and is responded by a single peer. 

### Query for the ledger height
```
node blk_height.js http://0.0.0.0:22000 
```

If successful, the console shall print:
```
The chain currently has <height> blocks. 
```
### Inspect the block by block number. 
Note, the last block number is (chain height - 1). 

```
node poll_block.js http://0.0.0.0:22000 1
```

If successful, the console shall print: 
```
Block 1 has the following structure: 
<block structure>
```

### Inspect a transaction by its hash
```
node poll_txn.js http://0.0.0.0:22003 <txn_hash>
```
\<txn_hash\> can either be subsitituted with the txn hash from the deployment transaction or from the invocation transaction. 

If successful, the console shall print: 
```
Txn <txn_hash> has the following structure: 
<Txn structure details>
```

## Spin off
```
docker-compose down
```
**NOTE**: future `PRIVATE_CONFIG=ignore docker-compose up -d` will be a fresh start. 

# Solidity Contract Q&A:
__Can we get txn information in the past block during contract execution__?

Unfortunately no. The solidity contract needs to be compiled into the opcode of Ethereum Virtual Machine. There are no opcodes to achieve this function. However, there exists a workaround via offchain `Oraclize`, detailed [here](https://stackoverflow.com/questions/50966458/can-we-get-transaction-information-recorded-in-the-past-block-using-solidity-in). 

__Can we repeatedly invoke a contract__?

Yes. Each valid invocation will result into a distinct transaction, which make effects only once. For example, if we invoke `add 10` **twice** on `simplestorage`, the `storedData` will be incremented by `20`. 
