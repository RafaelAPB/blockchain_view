# Before Execution
## Preparation
For each execution, we need to repeat commands in **Spin up the network**, **Create the channel** and **Join the channel** in [README.md](README.md). 
But we can reuse the crypto-materials and channel artifacts generated from commands in **Before Execution**.  

## Install and Instantiate a Smart Contract with privacy support 
### Copy the chaincode
Copy the chaincode *private_contract* under ${GOPATH}/src:
(This step need not repeat if we have already copied the chaincode file to `$GOPATH` and haven't modified go code in `$CC_PATH`. )
```
CC_PATH=private_contract # Path relative to the current directory
CC_NAME=$(basename ${CC_PATH})
rm -rf ${GOPATH}/src/${CC_PATH}
cp -r ${CC_PATH}/ ${GOPATH}/src/
CC_VERSION=1.0
```

### Install (Org1 ONLY)
```
export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_LOCALMSPID=Org1MSP 
export CORE_PEER_MSPCONFIGPATH=./crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp 

CC_PATH=private_contract # Path relative to the current directory
CC_NAME=$(basename ${CC_PATH})
CC_VERSION=1.0
./bin/peer chaincode install -n ${CC_NAME} -p ${CC_PATH} -v ${CC_VERSION}
```

### Instantiate
```
export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_LOCALMSPID=Org1MSP 
export CORE_PEER_MSPCONFIGPATH=./crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp 

./bin/peer chaincode instantiate -o ${ORDERER_ADDR} -C ${CHANNEL_NAME} -c '{"Args":["init"]}' -n ${CC_NAME}  -v ${CC_VERSION} -P "OR ('Org1MSP.member')" --collections-config private_contract/collections_config.json
```

## Install and Instantiate a Smart Contract for View management 

### Copy the chaincode
(As above, this step need not repeat if unnecessary. )
```
CC_PATH=view_storage # Path relative to the current directory
CC_NAME=$(basename ${CC_PATH})
CC_VERSION=1.0
rm -rf ${GOPATH}/src/${CC_PATH}
cp -r ${CC_PATH}/ ${GOPATH}/src/
```

### Install (Org1 and Org2 both)
For the peer of Org1:
```
export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_LOCALMSPID=Org1MSP 
export CORE_PEER_MSPCONFIGPATH=./crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp 

CC_PATH=view_storage # Path relative to the current directory
CC_NAME=$(basename ${CC_PATH})
CC_VERSION=1.0
./bin/peer chaincode install -n ${CC_NAME} -p ${CC_PATH} -v ${CC_VERSION}
```

For the peer of Org2:
```
export CORE_PEER_ADDRESS=localhost:8051
export CORE_PEER_LOCALMSPID=Org2MSP 
export CORE_PEER_MSPCONFIGPATH=./crypto_config/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp 

./bin/peer chaincode install -n ${CC_NAME} -p ${CC_PATH} -v ${CC_VERSION}
```

### Instantiate
```
export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_LOCALMSPID=Org1MSP 
export CORE_PEER_MSPCONFIGPATH=./crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp 

./bin/peer chaincode instantiate -o ${ORDERER_ADDR} -C ${CHANNEL_NAME} -c '{"Args":["init"]}' -n ${CC_NAME}  -v ${CC_VERSION} -P "OR ('Org1MSP.member', 'Org2MSP.member')"
```

# View Message Demo
## Assumption
We assume a transaction on the chain results from the invocation of a contract with public args and private args. 
The public args are public available on the chain. 
Private args are known to the invoking users only. 
Any other users, with the knowledge of private args, can recover the full information on the transaction.
Hence, the access control management is performed solely on the private args. 

In the following, a __view message__ is a map structure that associates the transaction IDs with its private args for all related transactions in this view. 

## Irrevocable View (Snapshot-based)
```
CallerKeyPath=$(ls crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore/*)
CallerCertPath=crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts/Admin@org1.example.com-cert.pem
CallerMsp=Org1MSP
CHANNEL_NAME=demo
Peer1Addr=grpc://localhost:7051
OrdererAddr=grpc://localhost:7050
CC_NAME=private_contract
View_CC_NAME=view_storage

node irrevocable_view_demo.js ${CallerKeyPath} ${CallerCertPath} ${CallerMsp} ${CHANNEL_NAME} ${Peer1Addr}  ${OrdererAddr} ${CC_NAME} ${View_CC_NAME}
```

The script demonstrates an end-to-end example for the prototype of the irrevocable view management.
* A user U1 invokes two transactions with public and private args. 
* U1 creates a view for this single transaction by encoding each key/value of the view message with a password and uploading the encoded association to the chain. 
* Afterwards U1 distributes view-specific password to a User U2. 
* Then User U2 pulls the encoded view message from the chain and recovers it with the provided password. 

Refer to the console display for the details of each step. 

## Revocable View (Delta-based)
```
CallerKeyPath=$(ls crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore/*)
CallerCertPath=crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts/Admin@org1.example.com-cert.pem
CallerMsp=Org1MSP
CHANNEL_NAME=demo
Peer1Addr=grpc://localhost:7051
OrdererAddr=grpc://localhost:7050
CC_NAME=private_contract
View_CC_NAME=view_storage

node revocable_view_demo.js ${CallerKeyPath} ${CallerCertPath} ${CallerMsp} ${CHANNEL_NAME} ${Peer1Addr}  ${OrdererAddr} ${CC_NAME} ${View_CC_NAME}
```

The script demonstrates an end-to-end example for the prototype of the revocable view management.

* A user U1 invokes a private transaction and creates a view for this single transaction by uploading the __hash__ of each key/value in the view message to the chain. 
* U1 again invokes another private transaction and **appends** this new transaction to the created view. 
* U1 then distributes the encoded view message, (encoded by a random password), which consists of two transactions, to a User U2. U1 also sends the encoded password protected by U2's public key. 
* Then User U2 recovers the password and then uses the password to recover the view message. U2 then pulls the key/value hash of the view message from the chain for the validation. 

Refer to the console display for the details of each step. 

# Post-Execution
## Spin off the network
```
docker-compose down
```

## Remove the chaincode images
This step can be skipped if chaincode/contract files are not modified. 

###  For View_storage Contract
```
docker rmi -f $(docker images --format '{{.Repository}}:{{.Tag}}' | grep 'view_storage')
```

###  For private_contract Contract
```
docker rmi -f $(docker images --format '{{.Repository}}:{{.Tag}}' | grep 'private_contract')
```








# All in All
```
docker-compose up -d

sleep 5s

export FABRIC_CFG_PATH=.
export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_LOCALMSPID=Org1MSP 
export CORE_PEER_MSPCONFIGPATH=./crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp 

ORDERER_ADDR=localhost:7050
CHANNEL_NAME=demo
./bin/peer channel create -o $ORDERER_ADDR -c $CHANNEL_NAME -f ./channel_artifacts/channel.tx


export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_LOCALMSPID=Org1MSP 
export CORE_PEER_MSPCONFIGPATH=./crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp 
CHANNEL_NAME=demo

./bin/peer channel join -b ./${CHANNEL_NAME}.block


export CORE_PEER_ADDRESS=localhost:8051
export CORE_PEER_LOCALMSPID=Org2MSP 
export CORE_PEER_MSPCONFIGPATH=./crypto_config/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp 
CHANNEL_NAME=demo

./bin/peer channel join -b ./${CHANNEL_NAME}.block

sleep 5s

# Install Private Contract
CC_PATH=private_contract # Path relative to the current directory
CC_NAME=$(basename ${CC_PATH})
rm -rf ${GOPATH}/src/${CC_PATH}
cp -r ${CC_PATH}/ ${GOPATH}/src/
CC_VERSION=1.0


export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_LOCALMSPID=Org1MSP 
export CORE_PEER_MSPCONFIGPATH=./crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp 

CC_PATH=private_contract # Path relative to the current directory
CC_NAME=$(basename ${CC_PATH})
CC_VERSION=1.0
./bin/peer chaincode install -n ${CC_NAME} -p ${CC_PATH} -v ${CC_VERSION}


export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_LOCALMSPID=Org1MSP 
export CORE_PEER_MSPCONFIGPATH=./crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp 

./bin/peer chaincode instantiate -o ${ORDERER_ADDR} -C ${CHANNEL_NAME} -c '{"Args":["init"]}' -n ${CC_NAME}  -v ${CC_VERSION} -P "OR ('Org1MSP.member')" --collections-config private_contract/collections_config.json


# Install View Manager Contract

CC_PATH=view_storage # Path relative to the current directory
CC_NAME=$(basename ${CC_PATH})
CC_VERSION=1.0
rm -rf ${GOPATH}/src/${CC_PATH}
cp -r ${CC_PATH}/ ${GOPATH}/src/


export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_LOCALMSPID=Org1MSP 
export CORE_PEER_MSPCONFIGPATH=./crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp 

CC_PATH=view_storage # Path relative to the current directory
CC_NAME=$(basename ${CC_PATH})
CC_VERSION=1.0
./bin/peer chaincode install -n ${CC_NAME} -p ${CC_PATH} -v ${CC_VERSION}



export CORE_PEER_ADDRESS=localhost:8051
export CORE_PEER_LOCALMSPID=Org2MSP 
export CORE_PEER_MSPCONFIGPATH=./crypto_config/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp 

./bin/peer chaincode install -n ${CC_NAME} -p ${CC_PATH} -v ${CC_VERSION}

export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_LOCALMSPID=Org1MSP 
export CORE_PEER_MSPCONFIGPATH=./crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp 

./bin/peer chaincode instantiate -o ${ORDERER_ADDR} -C ${CHANNEL_NAME} -c '{"Args":["init"]}' -n ${CC_NAME}  -v ${CC_VERSION} -P "OR ('Org1MSP.member', 'Org2MSP.member')"
```


```
CallerKeyPath=$(ls crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore/*)
CallerCertPath=crypto_config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts/Admin@org1.example.com-cert.pem
CallerMsp=Org1MSP
CHANNEL_NAME=demo
Peer1Addr=grpc://localhost:7051
OrdererAddr=grpc://localhost:7050
CC_NAME=private_contract
VIEW_CCID=view_storage

node view_demo.js ${CallerKeyPath} ${CallerCertPath} ${CallerMsp} ${CHANNEL_NAME} ${Peer1Addr}  ${OrdererAddr} ${CC_NAME} ${VIEW_CCID}
```


```
docker-compose down
```

# Presentation Schedule
Illustrate the distinction between encryption-based/hashed-based irrevocable/revocable view management. 
Revocable management does not send the encoded confidential data into the blockchain. 

Explain the FabricSupport.js with respect to view_storage.go and simplestorage.go

Explain the encryption-based revocable view management. 
* Explain one step, execute one step

Explain the hash-based revocable view management. 
* Explain one step, execute one step