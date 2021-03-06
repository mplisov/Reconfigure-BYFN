### To add the Org3 to the consortium and give it the authority to create a new channel it must be added to the system channel

### Once Org3 is added to the channel the below steps need to be run from cli container since Orderer is in that container

### Switch to Orderer

CORE_PEER_LOCALMSPID="OrdererMSP"
CORE_PEER_TLS_ROOTCERT_FILE=$ORDERER_CA
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp

### Fetch the genesis block for system channel
peer channel fetch config sys_config_block.pb -o orderer.example.com:7050 -c testchainid --tls --cafile $ORDERER_CA

### Decode the block to json format

curl -X POST --data-binary @sys_config_block.pb "$CONFIGTXLATOR_URL/protolator/decode/common.Block" | jq . > sys_config_block.json

### Isolating current config

jq .data.data[0].payload.data.config sys_config_block.json > sys_config.json

### Append and add org3.json and then write the output to sys_updated_config.json

jq -s '.[0] * {"channel_group":{"groups":{"Consortiums":{"groups": {"SampleConsortium": {"groups": {"Org3MSP":.[1]}}}}}}}' sys_config.json ./scripts/org3.json >& sys_updated_config.json

### Translating original config to proto

curl -X POST --data-binary @sys_config.json "$CONFIGTXLATOR_URL/protolator/encode/common.Config" > sys_config.pb

### Translating updated config to proto

curl -X POST --data-binary @sys_updated_config.json "$CONFIGTXLATOR_URL/protolator/encode/common.Config" > sys_updated_config.pb

### Computing config update

curl -X POST -F channel=testchainid -F "original=@sys_config.pb" -F "updated=@sys_updated_config.pb" "${CONFIGTXLATOR_URL}/configtxlator/compute/update-from-configs" > sys_config_update.pb

### Decoding config update

curl -X POST --data-binary @sys_config_update.pb "$CONFIGTXLATOR_URL/protolator/decode/common.ConfigUpdate" | jq . > sys_config_update.json

### Generating config update and wrapping it in an envelope

echo '{"payload":{"header":{"channel_header":{"channel_id":"testchainid", "type":2}},"data":{"config_update":'$(cat sys_config_update.json)'}}}' | jq . > sys_config_update_in_envelope.json

### Encoding config update envelope

curl -X POST --data-binary @sys_config_update_in_envelope.json "$CONFIGTXLATOR_URL/protolator/encode/common.Envelope" > sys_config_update_in_envelope.pb

### Switch to admin user of orderer

CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/users/Admin@example.com/msp

### Sending config update to orderer
peer channel update -f sys_config_update_in_envelope.pb -c testchainid -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

### To create a new channel the profile needs to be updated and defined as TwoOrgsChannel1 in the configtx.yaml file with only Org1 and Org3

### The below steps should be done outside of cli container in a separate terminal window from org3-artifacts dir

### Next we need to set the path for configtxgen

FABRIC_CFG_PATH=$PWD

### Generate channel configuration transaction

../../bin/configtxgen -profile NewOrgChannel -outputCreateChannelTx ../channel-artifacts/channel1.tx -channelID mychannel1

### Switch to Org3cli terminal and create a new channel with Org3

peer channel create -o orderer.example.com:7050 -c mychannel1 -f ./channel-artifacts/channel1.tx --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA

### Join peer0 of Org3 to the new channel

peer channel join -b mychannel1.block

### Install chaincode on peer of Org3

peer chaincode install -n newcc -v 1.0 -p github.com/chaincode/chaincode_example02/go/

### Instantiate the chaincode

peer chaincode instantiate -o orderer.example.com:7050 --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA -C mychannel1 -n newcc -v 1.0 -c '{"Args":["init","a","100","b","200"]}' -P "OR	('Org1MSP.member','Org3MSP.member')"

### Switch to Org1 in cli and fetch the latest config block

peer channel fetch config mychannel1.pb -o orderer.example.com:7050 -c mychannel1 --tls --cafile $ORDERER_CA

peer channel join -b mychannel1.pb

### Install chaincode on Org1

peer chaincode install -n newcc -v 1.0 -p github.com/chaincode/chaincode_example02/go/

### Make an invoke and query from peer of org3

peer chaincode invoke -o orderer.example.com:7050 --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA -C mychannel1 -n newcc -c '{"Args":["invoke","a","b","10"]}'

peer chaincode query -C mychannel1 -n newcc -c '{"Args":["query","a"]}'
