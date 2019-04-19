# Add Org
Adding an Org to a Channel While Network Run

## Clone Project

 ```
 $ git clone https://github.com/ChawinNewviyawong/AddOrg.git
 ```

## Setup Environment

1. generate certifacates and keys for base network entities
 ```
 $ ./generate.sh
 ```

2. copy filename *_sk from part `pwd`/crypto-config/peerOrganizations/org1.example.com/ca/ paste into docker-compose.yaml
 ```
    ...
    environment:
      - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
      - FABRIC_CA_SERVER_CA_NAME=ca1.example.com
      - FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org1.example.com-cert.pem
      - FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server-config/<paste *_sk>
    ...
 ```
   copy other ca filename to other org

3. start network

 ```
 $ ./start.sh
 ```

## Generate Org2 Crypto Material

 ```
 $ cd org2-artifacts
 $ ./generate-add-org.sh 2
 ```

 ``gererate-add-org.sh`` script:
 ```
 #!/bin/sh
 set -e

 ORG=$1

 # Generate the Org2 Crypto Material
 echo "================== Generate Org${ORG} Crypto Material =================="

 cd org${ORG}-artifacts

 ../../bin/cryptogen generate --config=./crypto-config-org2.yaml
 
 ../../bin/configtxgen -printOrg Org${ORG}MSP > ../channel-artifacts/org${ORG}.json

 cd ../ && cp -r crypto-config/ordererOrganizations org${ORG}-artifacts/crypto-config/
 ```

## Prepare CLI Environment

 1. execute to cli container
 ```
 $ docker exec -it cli bash
 ```
 2. Export ``CHANNEL_NAME`` variables:
 ```
 $ export CHANNEL_NAME=mychannel
 ```

## Fetch the Configuration

 ```
 $ peer channel fetch config config_block.pb -o orderer.example.com:7050 -c $CHANNEL_NAME
 ```

## Convert the Configuration to JSON and Trim It Down

 ```
 $ configtxlator proto_decode --input config_block.pb --type common.Block | jq .data.data[0].payload.data.config > config.json
 ```

## Add the Org2 Crypto Material

 ```
 $ jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org2MSP":.[1]}}}}}' config.json ./channel-artifacts/org2.json > modified_config.json
 ```

 ```
 $ configtxlator proto_encode --input config.json --type common.Config --output config.pb
 ```

 ```
 $ configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb
 ```

 ```
 $ configtxlator compute_update --channel_id $CHANNEL_NAME --original config.pb --updated modified_config.pb --output org2_update.pb
 ```

 ```
 $ configtxlator proto_decode --input org2_update.pb --type common.ConfigUpdate | jq . > org2_update.json
 ```

 ```
 $ echo '{"payload":{"header":{"channel_header":{"channel_id":"mychannel", "type":2}},"data":{"config_update":'$(cat org2_update.json)'}}}' | jq . > org2_update_in_envelope.json
 ```

 ```
 $ configtxlator proto_encode --input org2_update_in_envelope.json --type common.Envelope --output org2_update_in_envelope.pb
 ```
## Sign and Submit the Config Update

 ```
 $ peer channel signconfigtx -f org2_update_in_envelope.pb
 ```

 ```
 $ peer channel update -f org2_update_in_envelope.pb -c $CHANNEL_NAME -o orderer.example.com:7050
 ```

## Join Org2 to Channel

 ```
 $ docker-compose -f docker-compose-org2.yaml up -d
 ```

 ```
 $ docker exec -it Org2cli bash
 ```

 ```
 $ export CHANNEL_NAME=mychannel
 ```

 ```
 $ peer channel fetch 0 mychannel.block -o orderer.example.com:7050 -c $CHANNEL_NAME
 ```

 ```
 $ peer channel join -b mychannel.block
 ```

## Upgrade and Invoke Chaincode

 ```
 $ peer chaincode install -n <chaincode name> -v 2.0 -p github.com/chaincode/chaincode_example02/go/
 ```

 ```
 $ peer chaincode install -n <chaincode name> -v 2.0 -p github.com/chaincode/chaincode_example02/go/
 ```

 ```
 $ peer chaincode upgrade -o orderer.example.com:7050 -C $CHANNEL_NAME -n <chaincode name> -v 2.0 -c '{"Args":["init","a","90","b","210"]}' -P "OR ('Org1MSP.member','Org2MSP.member')"
 ```
