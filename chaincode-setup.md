## Product Trace Chaincode (trace-network2)

This README documents the end-to-end process to deploy and test the JavaScript chaincode named `product-trace` on the channel `tracechannel` for the 3-org topology:

* ManufacturerMSP: `peer0` (7051) and `peer1` (8051)
* DistributorMSP: `peer0` (9051) and `peer1` (10051)
* RetailerMSP: `peer0` (11051)
* Orderer (RAFT): `orderer1.example.com` (client endpoint `localhost:7050`)

### Chaincode

Location:

* `chaincode/product-trace-javascript/`

Exported functions (contract name: `ProductTrace`):

* `CreateProduct(ctx, id, name, manufacturer)`
* `ShipProduct(ctx, id, newOwner)`
* `ReceiveProduct(ctx, id)`
* `GetProduct(ctx, id)`
* `GetHistory(ctx, id)` (uses `ctx.stub.getHistoryForKey(id)`)

Asset model fields written to the ledger:

* `docType: "product"`
* `ID`, `Name`, `Manufacturer`, `Owner`, `Status`, `Timestamp`

#### Deterministic timestamps (important)

Fabric requires the endorsed proposal read/write set to match across endorsing peers. If you use `new Date()` / `Date.now()` directly, different endorsers may generate slightly different values and your transactions will fail with:

* `ProposalResponsePayloads do not match`

This chaincode uses `ctx.stub.getTxTimestamp()` so `Timestamp` is consistent across endorsers for a single proposal.

---
## 1) Preconditions

1. Network containers must be up (peers + orderers running).
2. You must be in this folder:

```bash
cd ~/fabric-dev/fabric-samples/trace-network2
```

3. Set Fabric CLI env vars (repeat each terminal / session):

```bash
export PATH="$PWD/../bin:$PATH"
unset FABRIC_CFG_PATH
export FABRIC_CFG_PATH="$PWD/compose/docker/peercfg"

export CHANNEL_NAME=tracechannel
export CC_NAME=product-trace
export CC_VERSION=1.0
```

4. Set orderer TLS env vars (used for package approve/commit and invokes):

```bash
export ORDERER_CA="$PWD/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem"
export ORDERER_ADDRESS=localhost:7050
export ORDERER_TLS_HOST=orderer1.example.com

export CORE_PEER_TLS_ENABLED=true
```

---
## 2) Initial Deployment (example with SEQUENCE=1)

If this is your first time deploying, set:

```bash
export SEQUENCE=1
```

### 2.1 Package the chaincode

```bash
peer lifecycle chaincode package ${CC_NAME}.tar.gz \
  --path "$PWD/chaincode/product-trace-javascript" \
  --lang node \
  --label ${CC_NAME}_${CC_VERSION}

export PACKAGE_ID=$(peer lifecycle chaincode calculatepackageid ${CC_NAME}.tar.gz)
echo "PACKAGE_ID=${PACKAGE_ID}"
```

### 2.2 Install on all peers (5 installs)

Manufacturer org peers:

```bash
# peer0.manufacturer (7051)
export CORE_PEER_LOCALMSPID=ManufacturerMSP
export CORE_PEER_MSPCONFIGPATH="$PWD/organizations/peerOrganizations/manufacturer.example.com/users/Admin@manufacturer.example.com/msp"
export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_TLS_ROOTCERT_FILE="$PWD/organizations/peerOrganizations/manufacturer.example.com/peers/peer0.manufacturer.example.com/tls/ca.crt"
peer lifecycle chaincode install ${CC_NAME}.tar.gz

# peer1.manufacturer (8051)
export CORE_PEER_ADDRESS=localhost:8051
export CORE_PEER_TLS_ROOTCERT_FILE="$PWD/organizations/peerOrganizations/manufacturer.example.com/peers/peer1.manufacturer.example.com/tls/ca.crt"
peer lifecycle chaincode install ${CC_NAME}.tar.gz
```

Distributor org peers:

```bash
# peer0.distributor (9051)
export CORE_PEER_LOCALMSPID=DistributorMSP
export CORE_PEER_MSPCONFIGPATH="$PWD/organizations/peerOrganizations/distributor.example.com/users/Admin@distributor.example.com/msp"
export CORE_PEER_ADDRESS=localhost:9051
export CORE_PEER_TLS_ROOTCERT_FILE="$PWD/organizations/peerOrganizations/distributor.example.com/peers/peer0.distributor.example.com/tls/ca.crt"
peer lifecycle chaincode install ${CC_NAME}.tar.gz

# peer1.distributor (10051)
export CORE_PEER_ADDRESS=localhost:10051
export CORE_PEER_TLS_ROOTCERT_FILE="$PWD/organizations/peerOrganizations/distributor.example.com/peers/peer1.distributor.example.com/tls/ca.crt"
peer lifecycle chaincode install ${CC_NAME}.tar.gz
```

Retailer org peer:

```bash
# peer0.retailer (11051)
export CORE_PEER_LOCALMSPID=RetailerMSP
export CORE_PEER_MSPCONFIGPATH="$PWD/organizations/peerOrganizations/retailer.example.com/users/Admin@retailer.example.com/msp"
export CORE_PEER_ADDRESS=localhost:11051
export CORE_PEER_TLS_ROOTCERT_FILE="$PWD/organizations/peerOrganizations/retailer.example.com/peers/peer0.retailer.example.com/tls/ca.crt"
peer lifecycle chaincode install ${CC_NAME}.tar.gz
```

### 2.3 Approve for each org (Manufacturer, Distributor, Retailer)

Run exactly once per org with the same `SEQUENCE` and `PACKAGE_ID`.

```bash
# ManufacturerMSP approval
export CORE_PEER_LOCALMSPID=ManufacturerMSP
export CORE_PEER_MSPCONFIGPATH="$PWD/organizations/peerOrganizations/manufacturer.example.com/users/Admin@manufacturer.example.com/msp"
export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_TLS_ROOTCERT_FILE="$PWD/organizations/peerOrganizations/manufacturer.example.com/peers/peer0.manufacturer.example.com/tls/ca.crt"
peer lifecycle chaincode approveformyorg \
  -o ${ORDERER_ADDRESS} --ordererTLSHostnameOverride ${ORDERER_TLS_HOST} \
  --tls --cafile "${ORDERER_CA}" \
  --channelID ${CHANNEL_NAME} --name ${CC_NAME} --version ${CC_VERSION} \
  --package-id ${PACKAGE_ID} --sequence ${SEQUENCE}

# DistributorMSP approval
export CORE_PEER_LOCALMSPID=DistributorMSP
export CORE_PEER_MSPCONFIGPATH="$PWD/organizations/peerOrganizations/distributor.example.com/users/Admin@distributor.example.com/msp"
export CORE_PEER_ADDRESS=localhost:9051
export CORE_PEER_TLS_ROOTCERT_FILE="$PWD/organizations/peerOrganizations/distributor.example.com/peers/peer0.distributor.example.com/tls/ca.crt"
peer lifecycle chaincode approveformyorg \
  -o ${ORDERER_ADDRESS} --ordererTLSHostnameOverride ${ORDERER_TLS_HOST} \
  --tls --cafile "${ORDERER_CA}" \
  --channelID ${CHANNEL_NAME} --name ${CC_NAME} --version ${CC_VERSION} \
  --package-id ${PACKAGE_ID} --sequence ${SEQUENCE}

# RetailerMSP approval
export CORE_PEER_LOCALMSPID=RetailerMSP
export CORE_PEER_MSPCONFIGPATH="$PWD/organizations/peerOrganizations/retailer.example.com/users/Admin@retailer.example.com/msp"
export CORE_PEER_ADDRESS=localhost:11051
export CORE_PEER_TLS_ROOTCERT_FILE="$PWD/organizations/peerOrganizations/retailer.example.com/peers/peer0.retailer.example.com/tls/ca.crt"
peer lifecycle chaincode approveformyorg \
  -o ${ORDERER_ADDRESS} --ordererTLSHostnameOverride ${ORDERER_TLS_HOST} \
  --tls --cafile "${ORDERER_CA}" \
  --channelID ${CHANNEL_NAME} --name ${CC_NAME} --version ${CC_VERSION} \
  --package-id ${PACKAGE_ID} --sequence ${SEQUENCE}
```

### 2.4 Commit the chaincode definition (SEQUENCE=1)

```bash
peer lifecycle chaincode commit \
  -o ${ORDERER_ADDRESS} --ordererTLSHostnameOverride ${ORDERER_TLS_HOST} \
  --tls --cafile "${ORDERER_CA}" \
  --channelID ${CHANNEL_NAME} --name ${CC_NAME} \
  --version ${CC_VERSION} --sequence ${SEQUENCE} \
  --peerAddresses localhost:7051 \
  --tlsRootCertFiles "$PWD/organizations/peerOrganizations/manufacturer.example.com/peers/peer0.manufacturer.example.com/tls/ca.crt" \
  --peerAddresses localhost:9051 \
  --tlsRootCertFiles "$PWD/organizations/peerOrganizations/distributor.example.com/peers/peer0.distributor.example.com/tls/ca.crt" \
  --peerAddresses localhost:11051 \
  --tlsRootCertFiles "$PWD/organizations/peerOrganizations/retailer.example.com/peers/peer0.retailer.example.com/tls/ca.crt"
```

### 2.5 Verify it is committed

```bash
peer lifecycle chaincode querycommitted --channelID ${CHANNEL_NAME} --name ${CC_NAME}
```

---
## 3) Test the chaincode (invoke/query)

Your invokes are endorsed by the channel endorsement policy. If a specific invoke fails with endorsement errors, re-run the invoke and include more peers in `--peerAddresses` until it meets the channel policy.

### 3.1 CreateProduct

```bash
peer chaincode invoke \
  -o ${ORDERER_ADDRESS} --ordererTLSHostnameOverride ${ORDERER_TLS_HOST} \
  --tls --cafile "${ORDERER_CA}" \
  -C ${CHANNEL_NAME} -n ${CC_NAME} \
  --peerAddresses localhost:7051 \
  --tlsRootCertFiles "$PWD/organizations/peerOrganizations/manufacturer.example.com/peers/peer0.manufacturer.example.com/tls/ca.crt" \
  --peerAddresses localhost:9051 \
  --tlsRootCertFiles "$PWD/organizations/peerOrganizations/distributor.example.com/peers/peer0.distributor.example.com/tls/ca.crt" \
  -c '{"Args":["CreateProduct","SKU-001","Organic Rice","ManufacturerOrg"]}'
```

### 3.2 GetProduct

```bash
export CORE_PEER_LOCALMSPID=ManufacturerMSP
export CORE_PEER_MSPCONFIGPATH="$PWD/organizations/peerOrganizations/manufacturer.example.com/users/Admin@manufacturer.example.com/msp"
export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_TLS_ROOTCERT_FILE="$PWD/organizations/peerOrganizations/manufacturer.example.com/peers/peer0.manufacturer.example.com/tls/ca.crt"

peer chaincode query -C ${CHANNEL_NAME} -n ${CC_NAME} \
  -c '{"Args":["GetProduct","SKU-001"]}'
```

### 3.3 ShipProduct

```bash
peer chaincode invoke \
  -o ${ORDERER_ADDRESS} --ordererTLSHostnameOverride ${ORDERER_TLS_HOST} \
  --tls --cafile "${ORDERER_CA}" \
  -C ${CHANNEL_NAME} -n ${CC_NAME} \
  --peerAddresses localhost:7051 \
  --tlsRootCertFiles "$PWD/organizations/peerOrganizations/manufacturer.example.com/peers/peer0.manufacturer.example.com/tls/ca.crt" \
  --peerAddresses localhost:9051 \
  --tlsRootCertFiles "$PWD/organizations/peerOrganizations/distributor.example.com/peers/peer0.distributor.example.com/tls/ca.crt" \
  -c '{"Args":["ShipProduct","SKU-001","DistributorOrg"]}'
```

### 3.4 ReceiveProduct

```bash
peer chaincode invoke \
  -o ${ORDERER_ADDRESS} --ordererTLSHostnameOverride ${ORDERER_TLS_HOST} \
  --tls --cafile "${ORDERER_CA}" \
  -C ${CHANNEL_NAME} -n ${CC_NAME} \
  --peerAddresses localhost:7051 \
  --tlsRootCertFiles "$PWD/organizations/peerOrganizations/manufacturer.example.com/peers/peer0.manufacturer.example.com/tls/ca.crt" \
  --peerAddresses localhost:9051 \
  --tlsRootCertFiles "$PWD/organizations/peerOrganizations/distributor.example.com/peers/peer0.distributor.example.com/tls/ca.crt" \
  -c '{"Args":["ReceiveProduct","SKU-001"]}'
```

### 3.5 GetHistory

```bash
peer chaincode query -C ${CHANNEL_NAME} -n ${CC_NAME} \
  -c '{"Args":["GetHistory","SKU-001"]}'
```

---
## 4) Stopping/closing laptop (what happens to ledger data?)

This is purely about Docker containers and Docker volumes for the ledger/state.

### Recommended: stop containers (do NOT delete volumes)

When you want to pause and resume tomorrow:

```bash
docker-compose \
  -f compose/compose-test-net.yaml \
  -f compose/docker/docker-compose-test-net.yaml \
  stop
```

Start tomorrow (continue with the same ledgers and committed chaincode definitions):

```bash
docker-compose \
  -f compose/compose-test-net.yaml \
  -f compose/docker/docker-compose-test-net.yaml \
  start
```

In this mode, you generally do NOT need to re-run:

* `osnadmin channel join`
* `peer channel join`
* chaincode `package/install/approve/commit`

### If you run `down -v` (deletes ledger volumes)

If you do:

```bash
docker-compose \
  -f compose/compose-test-net.yaml \
  -f compose/docker/docker-compose-test-net.yaml \
  down -v
```

then Docker volumes are removed, and ledger/channel state is wiped. You will need to recreate:

* channel genesis/block config and orderer join
* peer channel join
* chaincode lifecycle (deploy/commit again)

---
## 5) Tomorrow checklist (if you used `stop`, not `down -v`)

1. Start containers:

```bash
cd ~/fabric-dev/fabric-samples/trace-network2
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml start
```

2. Re-export your CLI environment:

```bash
export PATH="$PWD/../bin:$PATH"
unset FABRIC_CFG_PATH
export FABRIC_CFG_PATH="$PWD/compose/docker/peercfg"

export CHANNEL_NAME=tracechannel
export CC_NAME=product-trace
export CC_VERSION=1.0
export ORDERER_CA="$PWD/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem"
export ORDERER_ADDRESS=localhost:7050
export ORDERER_TLS_HOST=orderer1.example.com
export CORE_PEER_TLS_ENABLED=true
```

3. You can directly test with:

```bash
peer chaincode query -C tracechannel -n product-trace -c '{"Args":["GetHistory","SKU-001"]}'
```

---
## 6) If you change the chaincode: what to run again?

When you edit chaincode code, you must do a chaincode upgrade (new lifecycle sequence).

### What changes require upgrade?

* Any JavaScript code change (new logic in `lib/productTrace.js`) => upgrade required
* New chaincode function / updated contract logic => upgrade required

### What does NOT get wiped by upgrade?

* World state / assets already stored in the ledger remain as-is
* Old chaincode state is not deleted automatically

### Upgrade commands (repeat deployment, but bump `SEQUENCE`)

1. Update `SEQUENCE`:

```bash
export SEQUENCE=3   # example: increment by 1 from last committed
```

2. Repackage + recompute package id:

```bash
peer lifecycle chaincode package ${CC_NAME}.tar.gz \
  --path "$PWD/chaincode/product-trace-javascript" \
  --lang node \
  --label ${CC_NAME}_${CC_VERSION}

export PACKAGE_ID=$(peer lifecycle chaincode calculatepackageid ${CC_NAME}.tar.gz)
echo "PACKAGE_ID=${PACKAGE_ID}"
```

3. Re-install on ALL peers (5 installs) using the new `${CC_NAME}.tar.gz`
4. Re-approve from ALL orgs (3 approvals) using the new `${PACKAGE_ID}` and new `${SEQUENCE}`
5. Re-commit with the new `${SEQUENCE}`

After commit, retry the invoke/query tests.

### Quick sanity checks

Check committed version/sequence:

```bash
peer lifecycle chaincode querycommitted --channelID ${CHANNEL_NAME} --name ${CC_NAME}
```

If you see `ProposalResponsePayloads do not match` again after a chaincode change, it almost always means you reintroduced non-determinism (like using wall-clock time). Use `ctx.stub.getTxTimestamp()` or ensure deterministic outputs.

