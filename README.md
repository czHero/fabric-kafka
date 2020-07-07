sh run.sh start "{'hosts':{'host1':{'name':'host1','ip':'172.21.28.226','port':22,'user':'root','pass':'kaziyuan@123'},'host2':{'name':'host2','ip':'172.21.28.225','port':22,'user':'root','pass':'kaziyuan@123'},'host3':{'name':'host3','ip':'172.21.28.224','port':22,'user':'root','pass':'kaziyuan@123'}}}"


```
rm -rf channel-artifacts crypto-config
../bin/cryptogen generate --config=./crypto-config.yaml
mkdir channel-artifacts
../bin/configtxgen -profile KafkaOrdererGenesis -channelID byfn-sys-channel -outputBlock ./channel-artifacts/genesis.block

export CHANNEL_NAME=mychannel
../bin/configtxgen -profile MyChannel -outputCreateChannelTx ./channel-artifacts/${CHANNEL_NAME}_channel.tx -channelID $CHANNEL_NAME
../bin/configtxgen -profile MyChannel -outputAnchorPeersUpdate ./channel-artifacts/${CHANNEL_NAME}_Org1MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org1MSP
../bin/configtxgen -profile MyChannel -outputAnchorPeersUpdate ./channel-artifacts/${CHANNEL_NAME}_Org2MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org2MSP
../bin/configtxgen -profile MyChannel -outputAnchorPeersUpdate ./channel-artifacts/${CHANNEL_NAME}_Org3MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org3MSP

export IMAGE_TAG=latest
export COMPOSE_PROJECT_NAME=test

docker-compose -f docker-compose-host1.yaml down --volumes --remove-orphans
docker-compose -f docker-compose-host1.yaml up -d
docker-compose -f docker-compose-host2.yaml down --volumes --remove-orphans
docker-compose -f docker-compose-host2.yaml up -d
docker-compose -f docker-compose-host3.yaml up -d
docker-compose -f docker-compose-host3.yaml down --volumes --remove-orphans

docker exec -it cli bash

export CHANNEL_NAME=mychannel
peer channel create -o orderer0.orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/${CHANNEL_NAME}_channel.tx  --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/orderer.example.com/orderers/orderer0.orderer.example.com/msp/tlscacerts/tlsca.orderer.example.com-cert.pem
peer channel join -b ${CHANNEL_NAME}.block 
peer channel update -o orderer0.orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/${CHANNEL_NAME}_Org1MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/orderer.example.com/orderers/orderer0.orderer.example.com/msp/tlscacerts/tlsca.orderer.example.com-cert.pem
peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/sacc/
peer chaincode instantiate  -C $CHANNEL_NAME -n mycc -v 1.0 -c '{"Args":["a","10"]}'  -P  "OR ('Org1MSP.peer','Org2MSP.peer','Org3MSP.peer')" -o orderer0.orderer.example.com:7050  --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/orderer.example.com/orderers/orderer0.orderer.example.com/msp/tlscacerts/tlsca.orderer.example.com-cert.pem
peer chaincode list --instantiated  -C $CHANNEL_NAME 

CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
CORE_PEER_ADDRESS=peer0.org2.example.com:7051
CORE_PEER_LOCALMSPID="Org2MSP"

peer channel join -b ${CHANNEL_NAME}.block 
peer channel update -o orderer0.orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/${CHANNEL_NAME}_Org2MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/orderer.example.com/orderers/orderer0.orderer.example.com/msp/tlscacerts/tlsca.orderer.example.com-cert.pem
peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/sacc/
peer chaincode list --instantiated  -C $CHANNEL_NAME 

peer chaincode invoke -o orderer0.orderer.example.com:7050  -C $CHANNEL_NAME -n mycc  -c '{"Args":["set","a","22"]}' --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/orderer.example.com/orderers/orderer0.orderer.example.com/msp/tlscacerts/tlsca.orderer.example.com-cert.pem

peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
```