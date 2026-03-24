# Trace Network 2 (Manufacturer / Distributor / Retailer + 3 RAFT orderers)

This folder is a customized copy of Fabric sample `test-network`, adapted to this target topology:

- **Manufacturer org**: `peer0`, `peer1`
- **Distributor org**: `peer0`, `peer1`
- **Retailer org**: `peer0`
- **RAFT orderers**: `orderer1`, `orderer2`, `orderer3`
- **Channel**: `tracechannel`

All commands below assume you are in:

```bash
cd ~/fabric-dev/fabric-samples/trace-network2
```

---

## What was changed in files (summary)

### Crypto (cryptogen)
This repo variant uses split peer org crypto configs (not a single `crypto-config.yaml`):

- `organizations/cryptogen/crypto-config-org1.yaml` → **Manufacturer**, `Count: 2`
- `organizations/cryptogen/crypto-config-org2.yaml` → **Distributor**, `Count: 2`
- `organizations/cryptogen/crypto-config-org3.yaml` → **Retailer**, `Count: 1` (**new file**)
- `organizations/cryptogen/crypto-config-orderer.yaml` → **orderer1/2/3** (removed extra orderers)

### Channel config (configtx)
- `configtx/configtx.yaml`
  - Defines `ManufacturerMSP`, `DistributorMSP`, `RetailerMSP`
  - Sets **AnchorPeers**:
    - Manufacturer: `peer0.manufacturer.example.com:7051`
    - Distributor: `peer0.distributor.example.com:9051`
    - Retailer: `peer0.retailer.example.com:11051`
  - Sets **etcdraft Consenters**:
    - `orderer1.example.com:7050`
    - `orderer2.example.com:8050`
    - `orderer3.example.com:9050`

### Docker compose
Updated to run 3 orderers + 5 peers:

- `compose/compose-test-net.yaml`
- `compose/docker/docker-compose-test-net.yaml`

---

## Phase 6 — Generate crypto material (peers + orderers)

Set CLI tools:

```bash
export PATH=$PWD/../bin:$PATH
```

Optional cleanup (fresh crypto):

```bash
rm -rf organizations/peerOrganizations organizations/ordererOrganizations
```

Generate peer org crypto:

```bash
cryptogen generate --config=organizations/cryptogen/crypto-config-org1.yaml --output=organizations
cryptogen generate --config=organizations/cryptogen/crypto-config-org2.yaml --output=organizations
cryptogen generate --config=organizations/cryptogen/crypto-config-org3.yaml --output=organizations
```

Generate orderer crypto:

```bash
cryptogen generate --config=organizations/cryptogen/crypto-config-orderer.yaml --output=organizations
```

---

## Phase 7 — Generate channel block for `tracechannel`

`configtxgen` reads `configtx/configtx.yaml`, so point `FABRIC_CFG_PATH` there for this step only:

```bash
export FABRIC_CFG_PATH=$PWD/configtx
mkdir -p channel-artifacts

configtxgen -profile ChannelUsingRaft \
  -outputBlock channel-artifacts/tracechannel.block \
  -channelID tracechannel
```

---

## Phase 9 — Start the network (3 orderers + 5 peers)

Validate compose (optional but recommended):

```bash
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml config >/tmp/trace-compose-merged.yaml
wc -l /tmp/trace-compose-merged.yaml
```

Start containers:

```bash
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d
```

Check they are up:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

You should see:
- **Orderers**: `orderer1.example.com`, `orderer2.example.com`, `orderer3.example.com`
- **Peers**: `peer0/1.manufacturer.example.com`, `peer0/1.distributor.example.com`, `peer0.retailer.example.com`

---

## Phase 10 — Join orderers to the channel (channel participation)

Join `tracechannel` on each orderer using the admin endpoint.

```bash
osnadmin channel join \
  --channelID tracechannel \
  --config-block channel-artifacts/tracechannel.block \
  -o localhost:7053 \
  --ca-file organizations/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/ca.crt \
  --client-cert organizations/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/server.crt \
  --client-key organizations/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/server.key

osnadmin channel join \
  --channelID tracechannel \
  --config-block channel-artifacts/tracechannel.block \
  -o localhost:8053 \
  --ca-file organizations/ordererOrganizations/example.com/orderers/orderer2.example.com/tls/ca.crt \
  --client-cert organizations/ordererOrganizations/example.com/orderers/orderer2.example.com/tls/server.crt \
  --client-key organizations/ordererOrganizations/example.com/orderers/orderer2.example.com/tls/server.key

osnadmin channel join \
  --channelID tracechannel \
  --config-block channel-artifacts/tracechannel.block \
  -o localhost:9053 \
  --ca-file organizations/ordererOrganizations/example.com/orderers/orderer3.example.com/tls/ca.crt \
  --client-cert organizations/ordererOrganizations/example.com/orderers/orderer3.example.com/tls/server.crt \
  --client-key organizations/ordererOrganizations/example.com/orderers/orderer3.example.com/tls/server.key
```

Verify each orderer sees the channel:

```bash
osnadmin channel list -o localhost:7053 \
  --ca-file organizations/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/ca.crt \
  --client-cert organizations/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/server.crt \
  --client-key organizations/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/server.key
```

---

## Phase 10 — Join peers to the channel

Important: the `peer` CLI needs `core.yaml`. For this repo, use:

```bash
unset FABRIC_CFG_PATH
export FABRIC_CFG_PATH=$PWD/compose/docker/peercfg
ls $FABRIC_CFG_PATH/core.yaml
```

Then join each peer using the right MSP + address + TLS root cert.

```bash
export CORE_PEER_TLS_ENABLED=true
```

### Manufacturer

```bash
# peer0.manufacturer (7051)
export CORE_PEER_LOCALMSPID=ManufacturerMSP
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/manufacturer.example.com/users/Admin@manufacturer.example.com/msp
export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_TLS_ROOTCERT_FILE=$PWD/organizations/peerOrganizations/manufacturer.example.com/peers/peer0.manufacturer.example.com/tls/ca.crt
peer channel join -b channel-artifacts/tracechannel.block

# peer1.manufacturer (8051)
export CORE_PEER_LOCALMSPID=ManufacturerMSP
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/manufacturer.example.com/users/Admin@manufacturer.example.com/msp
export CORE_PEER_ADDRESS=localhost:8051
export CORE_PEER_TLS_ROOTCERT_FILE=$PWD/organizations/peerOrganizations/manufacturer.example.com/peers/peer1.manufacturer.example.com/tls/ca.crt
peer channel join -b channel-artifacts/tracechannel.block
```

### Distributor

```bash
# peer0.distributor (9051)
export CORE_PEER_LOCALMSPID=DistributorMSP
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/distributor.example.com/users/Admin@distributor.example.com/msp
export CORE_PEER_ADDRESS=localhost:9051
export CORE_PEER_TLS_ROOTCERT_FILE=$PWD/organizations/peerOrganizations/distributor.example.com/peers/peer0.distributor.example.com/tls/ca.crt
peer channel join -b channel-artifacts/tracechannel.block

# peer1.distributor (10051)
export CORE_PEER_LOCALMSPID=DistributorMSP
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/distributor.example.com/users/Admin@distributor.example.com/msp
export CORE_PEER_ADDRESS=localhost:10051
export CORE_PEER_TLS_ROOTCERT_FILE=$PWD/organizations/peerOrganizations/distributor.example.com/peers/peer1.distributor.example.com/tls/ca.crt
peer channel join -b channel-artifacts/tracechannel.block
```

### Retailer

```bash
# peer0.retailer (11051)
export CORE_PEER_LOCALMSPID=RetailerMSP
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/retailer.example.com/users/Admin@retailer.example.com/msp
export CORE_PEER_ADDRESS=localhost:11051
export CORE_PEER_TLS_ROOTCERT_FILE=$PWD/organizations/peerOrganizations/retailer.example.com/peers/peer0.retailer.example.com/tls/ca.crt
peer channel join -b channel-artifacts/tracechannel.block
```

Quick verify (run per peer after exporting env for that peer):

```bash
peer channel list
```

---

## How to inspect the running network

### What containers are running / since when
```bash
docker ps --format "table {{.Names}}\t{{.RunningFor}}\t{{.Status}}\t{{.Ports}}"
```

### What channels each orderer has
```bash
osnadmin channel list -o localhost:7053 \
  --ca-file organizations/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/ca.crt \
  --client-cert organizations/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/server.crt \
  --client-key organizations/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/server.key
```

### What channel(s) a peer has joined
(Set the peer env like in the join section, then:)

```bash
peer channel list
```

---

## Stop / start tomorrow (two common options)

### Option A — Stop and resume later (keep all ledgers/channel membership)
This is the usual “pause and continue tomorrow”.

Stop containers **without deleting volumes**:

```bash
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml stop
```

Start them again:

```bash
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml start
```

In this option:
- You **do not** need to re-run `cryptogen`, `configtxgen`, `osnadmin join`, or `peer channel join`.
- The channel and ledgers are preserved in Docker volumes.

### Option B — Tear down and recreate from scratch (clean reset)
Use this if you want a clean environment (it deletes volumes/ledgers).

```bash
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml down -v
rm -rf channel-artifacts system-genesis-block
```

Then redo:
- Phase 6 (crypto) if you also removed `organizations/...`
- Phase 7 (regenerate `tracechannel.block`)
- Phase 9 (up)
- Phase 10 (orderer joins + peer joins)

---

## Troubleshooting notes (common gotchas)

### `peer` CLI: “core config file not found”
If you see:

> `Config File "core" Not Found ...`

you likely still have:

```bash
export FABRIC_CFG_PATH=$PWD/configtx
```

Fix by using:

```bash
unset FABRIC_CFG_PATH
export FABRIC_CFG_PATH=$PWD/compose/docker/peercfg
```

### Orderers exiting / TLS handshake errors / `system-channel` replication panic
If orderers exit and logs mention `system-channel` replication, it’s usually old ledger state.

Fix with a clean orderer reset:

```bash
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml down -v
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d
```

