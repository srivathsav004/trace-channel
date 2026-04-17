# Beckn Protocol Setup Guide for OATS

This guide will help you set up the Beckn protocol layer on top of your existing Hyperledger Fabric networks (oats-network1 and oats-network2).

## Prerequisites

1. **Node.js >= 18** installed
2. **Hyperledger Fabric networks running**:
   - oats-network1 (TRST01 organization)
   - oats-network2 (TRST01 organization)
3. **Existing OATS chaincode deployed** on both networks
4. **Backend API running** on port 3000 (from oats-network1/backend)

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Beckn Protocol Layer (L3)               │
├─────────────────────────────────────────────────────────────┤
│  BAP (Port 3001)                 BPP (Port 3002)            │
│  - Buyer/Searcher side            - Provider/Seller side     │
│  - search, select, init, confirm  - on_search, on_confirm   │
│                                    - on_status, on_update    │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Fabric Backend API (Port 3000)                │
├─────────────────────────────────────────────────────────────┤
│  - Translates HTTP requests to Fabric CLI commands          │
│  - Invokes chaincode functions                              │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Hyperledger Fabric Networks                    │
├─────────────────────────────────────────────────────────────┤
│  oats-network1 (TRST01)  │  oats-network2 (TRST01)         │
│  - oats-traceability chaincode                              │
│  - oatschannel                                             │
└─────────────────────────────────────────────────────────────┘
```

## Step-by-Step Setup

### Step 1: Integrate Beckn Chaincode Functions

The chaincode extensions in `chaincode-extensions/beckn-functions.js` need to be integrated into your existing chaincode.

**Action Required:**
1. Open `oats-network1/chaincode/oats-traceability-javascript/lib/oatsTraceability.js`
2. Add the functions from `chaincode-extensions/beckn-functions.js` to the `OATSTraceabilityContract` class
3. Rebuild and redeploy the chaincode:

```bash
cd oats-network1
./network.sh deployCC -ccn oats-traceability -ccp ../chaincode/oats-traceability-javascript -ccl javascript -ccv 1.1 -ccs 1
```

**Functions to add:**
- `declareLot()` - Genesis event for lot declaration
- `getLotsByFilter()` - Query lots with filters
- `transferCustody()` - Custody transfer between actors
- `dispatchLot()` - Record physical dispatch
- `receiveLot()` - Record physical receipt
- `transformLot()` - Processing/transformation events
- `splitMergeLot()` - Split/merge operations
- `getLotWithLineage()` - Get lot with parent-child lineage
- `generateDigitalSignature()` - Helper for digital signatures

### Step 2: Set Up BAP (Buyer Side)

1. **Navigate to BAP directory:**
   ```bash
   cd beckn-protocol/bap
   ```

2. **Install dependencies:**
   ```bash
   npm install
   ```

3. **Configure environment:**
   ```bash
   cp .env.example .env
   ```

4. **Edit `.env` file:**
   ```env
   PORT=3001
   BPP_URL=http://localhost:3002
   BACKEND_API_URL=http://localhost:3000
   FABRIC_NETWORK=oats-network1
   CHANNEL_NAME=oatschannel
   CHAINCODE_NAME=oats-traceability
   PEER_ADDRESS=localhost:7051
   ```

5. **Start BAP service:**
   ```bash
   npm start
   ```

### Step 3: Set Up BPP (Provider Side)

1. **Navigate to BPP directory:**
   ```bash
   cd beckn-protocol/bpp
   ```

2. **Install dependencies:**
   ```bash
   npm install
   ```

3. **Configure environment:**
   ```bash
   cp .env.example .env
   ```

4. **Edit `.env` file:**
   ```env
   PORT=3002
   BAP_URL=http://localhost:3001
   BACKEND_API_URL=http://localhost:3000
   FABRIC_NETWORK=oats-network1
   CHANNEL_NAME=oatschannel
   CHAINCODE_NAME=oats-traceability
   PEER_ADDRESS=localhost:7051
   ```

5. **Start BPP service:**
   ```bash
   npm start
   ```

### Step 4: Ensure Fabric Backend is Running

The BPP service calls the existing Fabric backend API. Make sure it's running:

```bash
cd oats-network1/backend
npm start
```

The backend should be running on port 3000.

### Step 5: Test the Integration

#### Test 1: Health Checks

```bash
# Check BAP health
curl http://localhost:3001/health

# Check BPP health
curl http://localhost:3002/health

# Check Fabric backend health
curl http://localhost:3000/health
```

#### Test 2: Search for Lots (Discovery)

```bash
curl -X POST http://localhost:3001/search \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "country": "IND",
      "city": "std:080",
      "action": "search",
      "timestamp": "2024-01-15T10:00:00Z",
      "bap_id": "did:example:bap",
      "bpp_id": "did:example:bpp"
    },
    "message": {
      "intent": {
        "commoditySector": "agriculture",
        "complianceStatus": "certified"
      }
    }
  }'
```

#### Test 3: Declare a Lot (Genesis Event)

First, you need to declare a lot using the Fabric backend directly:

```bash
curl -X POST http://localhost:3000/assets/invoke \
  -H "Content-Type: application/json" \
  -d '{
    "functionName": "declareLot",
    "args": [
      "SGTIN-001",
      "did:example:farmer1",
      "cotton",
      "agriculture",
      "GLN-001",
      "2024-01-15T00:00:00Z",
      "hash123"
    ]
  }'
```

#### Test 4: Transfer Custody (Approve Event)

```bash
curl -X POST http://localhost:3002/on_confirm \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "country": "IND",
      "city": "std:080",
      "action": "on_confirm",
      "timestamp": "2024-01-15T10:00:00Z",
      "bap_id": "did:example:bap",
      "bpp_id": "did:example:bpp"
    },
    "message": {
      "order": {
        "id": "ORDER_001",
        "items": [{
          "id": "SGTIN-001"
        }],
        "provider": {
          "id": "did:example:farmer1"
        },
        "consumer": {
          "id": "did:example:buyer1"
        },
        "quote": {
          "id": "QUOTE_001"
        }
      }
    }
  }'
```

#### Test 5: Dispatch Lot

```bash
curl -X POST http://localhost:3002/on_status \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "country": "IND",
      "city": "std:080",
      "action": "on_status",
      "timestamp": "2024-01-15T10:00:00Z",
      "bap_id": "did:example:bap",
      "bpp_id": "did:example:bpp"
    },
    "message": {
      "order": {
        "id": "ORDER_001",
        "items": [{
          "id": "SGTIN-001"
        }],
        "state": "InTransit",
        "gps_coords": "12.9716,77.5946",
        "transporter_id": "did:example:transporter1",
        "vehicle_id": "VEH001"
      }
    }
  }'
```

#### Test 6: Receive Lot

```bash
curl -X POST http://localhost:3002/on_status \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "country": "IND",
      "city": "std:080",
      "action": "on_status",
      "timestamp": "2024-01-15T10:00:00Z",
      "bap_id": "did:example:bap",
      "bpp_id": "did:example:bpp"
    },
    "message": {
      "order": {
        "id": "ORDER_001",
        "items": [{
          "id": "SGTIN-001"
        }],
        "state": "Delivered",
        "consignee_id": "did:example:consignee1",
        "condition_hash": "hash456"
      }
    }
  }'
```

## Protocol Event Mapping Reference

| Protocol Event | EPCIS 2.0 Event | Beckn API | Chaincode Function | OATS Layer |
|----------------|-----------------|-----------|-------------------|------------|
| Declare | ObjectEvent (ADD) | Actor-initiated | declareLot() | L1 + L2 |
| Discovery | QueryEvent | search / on_search | getLotsByFilter() | L3 |
| Request | TransactionEvent (POSS_XFER) | select / init / confirm | (endorsement) | L2 + L3 |
| Approve | TransactionEvent (XFER) | on_confirm | transferCustody() | L2 + L3 |
| Dispatch | ObjectEvent (DEPART) | on_status | dispatchLot() | L2 + L3 |
| Receipt | ObjectEvent (ARRIVE) | on_status | receiveLot() | L2 |
| Change of State | TransformationEvent | update / on_update | transformLot() | L2 + L6 |
| Change of State | AggregationEvent | update / on_update | splitMergeLot() | L2 + L6 |

## Troubleshooting

### Issue: BAP/BPP cannot connect to Fabric backend

**Solution:** Ensure the Fabric backend is running on port 3000:
```bash
cd oats-network1/backend
npm start
```

### Issue: Chaincode function not found

**Solution:** Ensure you've integrated the Beckn chaincode functions and redeployed the chaincode. Check the chaincode version.

### Issue: Port already in use

**Solution:** Change the PORT in the `.env` file for BAP or BPP if the default ports (3001, 3002) are already in use.

### Issue: Connection refused between BAP and BPP

**Solution:** Ensure both services are running and the URLs in `.env` files are correct.

## Next Steps

1. **Integrate with actual Beckn Registry:** The current implementation uses direct BAP-BPP communication. For production, integrate with a Beckn Registry for discovery.

2. **Add Authentication:** Implement proper authentication and authorization for BAP and BPP endpoints.

3. **Add Digital Signatures:** Enhance the `generateDigitalSignature()` helper function with proper cryptographic signing.

4. **Add DID Resolution:** Implement proper DID resolution for actor identities.

5. **Add EPCIS Event Generation:** Generate proper EPCIS 2.0 XML/JSON events for each protocol event.

6. **Add Webhook Support:** Implement webhook support for asynchronous Beckn protocol communication.

7. **Deploy to Production:** Deploy BAP and BPP services to your VMs alongside the Fabric networks.

## Support

For issues or questions:
1. Check the logs in the BAP/BPP consoles
2. Verify Fabric backend logs
3. Check chaincode logs using `docker logs`
4. Review the protocol event mapping to ensure correct function calls
