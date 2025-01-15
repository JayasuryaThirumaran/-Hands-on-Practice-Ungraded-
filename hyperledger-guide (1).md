# Hyperledger Fabric: Adding a New Peer to Existing Organization

## Overview
This exercise guides you through the process of adding a new peer (peer1) to an existing organization (Org1) in a running Hyperledger Fabric network. You'll learn how to generate crypto material, configure the peer, and join it to existing channels.

## Prerequisites
- Running Hyperledger Fabric network (test-network)
- Docker and Docker Compose
- Fabric binaries and Docker images
- Github Codespaces (if applicable)

## Tasks

### 1. Initial Setup Verification

First, verify your existing network:

```bash
cd fabric-samples/test-network
docker ps
```

Expected output should show existing peers (peer0.org1, peer0.org2) and orderer running.

### 2. Generate Crypto Material for New Peer

Create a crypto configuration for the new peer. Create `peer1-crypto.yaml`:

```yaml
PeerOrgs:
  - Name: Org1
    Domain: org1.example.com
    EnableNodeOUs: true
    Specs:
      - Hostname: peer1
```

Generate the crypto material:

```bash
../bin/cryptogen extend --config=peer1-crypto.yaml --input="organizations"
```

### 3. Create Peer Configuration

Create a directory for the new peer:

```bash
mkdir -p docker/peer1.org1.example.com
```

Create `docker/peer1.org1.example.com/core.yaml`:

```yaml
peer:
  id: peer1.org1.example.com
  networkId: dev
  listenAddress: 0.0.0.0:7151
  address: peer1.org1.example.com:7151
  chaincodeAddress: peer1.org1.example.com:7152
  chaincodeListenAddress: 0.0.0.0:7152

vm:
  endpoint: unix:///host/var/run/docker.sock

chaincode:
  builder: $(DOCKER_NS)/fabric-ccenv:$(TWO_DIGIT_VERSION)
  pull: false

ledger:
  state:
    stateDatabase: goleveldb

operations:
  listenAddress: 127.0.0.1:9444
```

### 4. Update Docker Compose Configuration

Create `docker/docker-compose-peer1.yaml`:

```yaml
version: '3.7'

networks:
  test:

services:
  peer1.org1.example.com:
    container_name: peer1.org1.example.com
    image: hyperledger/fabric-peer:latest
    environment:
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=fabric_test
      - FABRIC_LOGGING_SPEC=INFO
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_PROFILE_ENABLED=false
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
      - CORE_PEER_ID=peer1.org1.example.com
      - CORE_PEER_ADDRESS=peer1.org1.example.com:7151
      - CORE_PEER_LISTENADDRESS=0.0.0.0:7151
      - CORE_PEER_CHAINCODEADDRESS=peer1.org1.example.com:7152
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:7152
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org1.example.com:7051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1.org1.example.com:7151
      - CORE_PEER_LOCALMSPID=Org1MSP
    volumes:
      - /var/run/docker.sock:/host/var/run/docker.sock
      - ../organizations/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/msp:/etc/hyperledger/fabric/msp
      - ../organizations/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls:/etc/hyperledger/fabric/tls
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
    networks:
      - test
    extra_hosts:
      - "orderer.example.com:172.17.0.1"
      - "peer0.org1.example.com:172.17.0.1"
      - "peer0.org2.example.com:172.17.0.1"
```

### 5. Start the New Peer

```bash
docker-compose -f docker/docker-compose-peer1.yaml up -d
```

### 6. Join Channel

Set environment variables:

```bash
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=localhost:7151
```

Fetch the genesis block and join the channel:

```bash
peer channel fetch 0 ./channel-artifacts/mychannel.block -o localhost:7050 \
    --ordererTLSHostnameOverride orderer.example.com \
    -c mychannel \
    --tls \
    --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

peer channel join -b ./channel-artifacts/mychannel.block
```

### 7. Verify Peer Status

```bash
peer channel list
peer channel getinfo -c mychannel
```

## Expected Outcomes

After completing this exercise, you should:
- Have a new peer running in the network
- Successfully joined the peer to the channel
- Be able to verify peer operation

## Verification Tasks
1. Check if the peer container is running:
   ```bash
   docker ps | grep peer1.org1
   ```

2. Check peer logs:
   ```bash
   docker logs peer1.org1.example.com
   ```

3. Verify channel membership:
   ```bash
   peer channel list
   ```

## Submission Requirements (Optional)

1. Create a repository in your forked github page as 'Screenshots' and add screenshots of:
   - Docker container status
   - Channel join confirmation
   - Channel list output
2. Brief description of any challenges encountered
3. Logs showing successful peer operation

## Common Issues and Solutions

1. **TLS Connection Failure**
   - Verify TLS certificate paths
   - Check if certificates were generated correctly
   - Ensure hostname resolution is correct

2. **Channel Join Failure**
   - Verify genesis block fetch was successful
   - Check peer environment variables
   - Ensure orderer is accessible

3. **Peer Start-up Issues**
   - Check Docker logs for errors
   - Verify port availability
   - Ensure crypto material is in correct location

## Tips
- Always verify the network status before adding new components
- Keep logs for troubleshooting
- Test connectivity between components
- Backup existing crypto material before making changes

Need help? Refer to the [Hyperledger Fabric documentation](https://hyperledger-fabric.readthedocs.io/)