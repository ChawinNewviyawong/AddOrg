# Add Org
Adding an Org to a Channel While Network Run

## Clone Project

 ```
 $ git clone https://github.com/ChawinNewviyawong/AddOrg.git
 ```

## Setup Environment

 #### 1. generate certifacates and keys for base network entities
 ```
 $ ./generate.sh
 ```

 #### 2. copy ca filename *_sk from part `pwd`/crypto-config/peerOrganizations/org1.example.com/ca/ paste into docker-compose.yaml
   <br>
   and copy other ca filename to other org

  ```
  ...
  environment:
    - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
    - FABRIC_CA_SERVER_CA_NAME=ca1.example.com
    - FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org1.example.com-cert.pem
    - FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server-config/<paste *_sk>
  ...
  ```

 #### 3. start network

 ```
 $ ./start.sh
 ```

## Generate Org3 Crypto Material

 ```
 $ cd org3-artifacts
 ```
 #### 1. generate Org3 crypto material

 ```
 $ ../../bin/cryptogen generate --config=./crypto-config-org2.yaml
 ```
 this command to generate key and certificates for Org3

 #### 2. use configtxgen to print org3-specific configuration material in json
 
 ```
 $ ../../bin/configtxgen -printOrg Org3MSP > ../channel-artifacts/org3.json
 ```
 create json file to `channel-artifacts`. json file have policy, admin certificate, CA root cert, and TLS root cert.

 #### 3. copy Orderer Org's MSP to Org3 `crypto-config` directory
 ```
 $ cd ../
 $ cp -r crypto-config/ordererOrganizations org3-artifacts/crypto-config/
 ```
 for secure communication between Org3 and Orderer


## Prepare CLI Environment

 #### 1. execute to cli contrainer
 ```
 $ docker exec -it cli bash
 ```
 #### 2. export `CHANNEL_NAME` variables
 ```
 $ export CHANNEL_NAME=mychannel
 ```

 #### 3. fetch configuration
 ```
 $ peer channel fetch config config_block.pb -o orderer.example.com:7050 -c CHANNEL_NAME
 ```
 save channel configuration block to config_block.pb.

 #### 4. convert configuration to json and trim
 ```
 $ configtxlator proto_decode --input config_block.pb --type common.Block | jq .data.data[0].payload.data.config > config.json
 ```
 use `configtxlator` to decode config_block.pb to json format

 #### 5. add the Org3 crypto material
 ```
 $ jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org3MSP":.[1]}}}}}' config.json ./channel-artifacts/org3.json > modified_config.json
 ```

 ```
 $ configtxlator proto_encode --input config.json --type common.Config --output config.pb
 ```
 encode config.json to config.pb

 ```
 $ configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb
 ```
 encode modified_config.json to modified_config.pb

 ```
 $ configtxlator compute_update --channel_id $CHANNEL_NAME --original config.pb --updated modified_config.pb --output org3_update.pb
 ```
 update `config.pb` by `modified_config.pb` and create to `org3_update.pb` 

 ```
 $ configtxlator proto_decode --input org3_update.pb --type common.ConfigUpdate | jq . > org3_update.json
 ```
 decode `org3_update.pb` to `org3_update.json` into edit

 ```
 $ echo '{"payload":{"header":{"channel_header":{"channel_id":"mychannel", "type":2}},"data":{"config_update":'$(cat org3_update.json)'}}}' | jq . > org3_update_in_envelope.json
 ```
 give header field back that we stripped away earlier.

 ```
 $ configtxlator proto_encode --input org3_update_in_envelope.json --type common.Envelope --output org3_update_in_envelope.pb
 ```
 encode `org3_update_in_envelope.json` to `org3_update_in_envelope.pb`

 #### 6. sign and submit the config update
 However, we need signatures from the requisite Admin users before the config can be written to the ledger. we have only two orgs and the majority of two is two, we need both of them to sign. Without both signatures, the ordering service will reject the transaction for failing to fulfill the policy.
 ```
 $ peer channel signconfigtx -f org3_update_in_envelope.pb
 ```
 Org1 admin sign this update protobuf
 <br><br>
 export the Org2 environment variables
 ```
 export CORE_PEER_LOCALMSPID="Org2MSP"
 export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
 export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
 export CORE_PEER_ADDRESS=peer0.org2.example.com:8051
 ```

 ```
 $ peer channel update -f org3_update_in_envelope.pb -c $CHANNEL_NAME -o orderer.example.com:7050
 ```
 Org2 admin signature will be attached to this call so there is no need to manually sign the protobuf a second time

## Join Org3 to Channel

 #### 1. open new terminal, start Org3 docker compose
 ```
 $ docker-compose -f docker-compose-org3.yaml up -d
 ```
 #### 2. exec into the Org3 CLI container
 ```
 $ docker exec -it Org3cli bash
 ```
 #### 3. export `CHANNEL_NAME` variables 
 ```
 $ export CHANNEL_NAME=mychannel
 ```
 #### 4. fetch mychannel.block
 ```
 $ peer channel fetch 0 mychannel.block -o orderer.example.com:7050 -c CHANNEL_NAME
 ```
 #### 5. join mychannel.block
 ```
 $ peer channel join -b mychannel.block
 ```
 if you want to join other peer for Org3, export `ADDRESS` variables
 ```
 $ export CORE_PEER_ADDRESS=peer1.org3.example.com:9051
 $ peer channel join -b mychannel.block
 ```

## Install and Upgrade Chaincode

 #### 1.install chaincode in Org3 by Org3cli

 ```
 $ peer chaincode install -n <chaincode name> -v 2.0 -p github.com/chaincode/chaincode_example02/go/
 ```

 #### 2.install chaincode in Org1

 ```
 $ peer chaincode install -n <chaincode name> -v 2.0 -p github.com/chaincode/chaincode_example02/go/
 ```
 #### 3.flip to the Org2 and install chaincode again
 ```
 $ export CORE_PEER_LOCALMSPID="Org2MSP"
 $ export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
 $ export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
 $ export CORE_PEER_ADDRESS=peer0.org2.example.com:8051

 $ peer chaincode install -n <chaincode name> -v 2.0 -p github.com/chaincode/chaincode_example02/go/
 ```
 if you want to install chaincode other Org, flip to other Org
 <br>
 #### 4. upgrade chaincode
 ```
 $ peer chaincode upgrade -o orderer.example.com:7050 -C $CHANNEL_NAME -n <chaincode name> -v 2.0 -c '{"Args":["init","a","90","b","210"]}' -P "OR ('Org1MSP.member','Org2MSP.member','Org3.MSP.member')"
 ```
