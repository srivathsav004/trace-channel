# OATS + Beckn Protocol Complete Flow Guide

This guide explains the complete integration between the existing OATS traceability chaincode and the new Beckn protocol functions, including step-by-step curl examples.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Chaincode Structure](#chaincode-structure)
3. [Status Flow](#status-flow)
4. [Integration Flow](#integration-flow)
5. [Complete End-to-End Example](#complete-end-to-end-example)
6. [Curl Commands by Flow](#curl-commands-by-flow)

---

## Architecture Overview

### System Components

```
┌─────────────────────────────────────────────────────────────────┐
│                     Beckn Protocol Layer (L3)                 │
├─────────────────────────────────────────────────────────────────┤
│  BAP (Port 3001)                 BPP (Port 3002)               │
│  - search, select, init, confirm  - on_search, on_confirm      │
│  - status, update                 - on_status, on_update        │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              Fabric Backend API (Port 3000)                    │
├─────────────────────────────────────────────────────────────────┤
│  - Translates HTTP to Fabric CLI                               │
│  - /query/custom - Query functions                             │
│  - /query/invoke - Transaction functions                       │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│           Hyperledger Fabric Network (oatschannel)            │
├─────────────────────────────────────────────────────────────────┤
│  Chaincode: oats-traceability (JavaScript)                     │
│  Organization: TRST01                                           │
│  Peers: peer0.trst01.example.com (7051)                        │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
Client Request → BAP → BPP → Fabric Backend → Chaincode → Fabric Ledger
     ↓              ↓      ↓           ↓              ↓            ↓
  (JSON)        (JSON) (JSON)      (CLI)         (Smart Contract) (State DB)
```

---

## Chaincode Structure

### Existing OATS Chaincode Functions

The `oatsTraceability.js` chaincode contains these core functions:

#### Actor Management
- `CreateActor()` - Register a supply chain actor (farmer, buyer, transporter)
- `GetActor()` - Retrieve actor details
- `UpdateActorStatus()` - Update actor status

#### Asset (Lot) Management
- `DeclareAsset()` - Register a new asset/product lot
- `GetAsset()` - Retrieve asset details
- `UpdateAssetQuantity()` - Update asset quantity

#### Facility Management
- `CreateFacility()` - Register a facility
- `GetFacility()` - Retrieve facility details

#### Traceability Events
- `CreateTraceabilityEvent()` - Record traceability events
- `GetTraceabilityEvent()` - Retrieve event details

#### Transformation
- `CreateTransformation()` - Record processing events
- `GetTransformation()` - Retrieve transformation details

#### Query Functions
- `GetAssetHistory()` - Get asset modification history
- `GetAssetLineage()` - Get parent-child lineage
- `QueryAssetsByActor()` - Query assets by owner
- `QueryEventsByAsset()` - Query events for an asset
- `GetSupplyChainTraceability()` - Get complete traceability report

### New Beckn Protocol Functions

The Beckn functions extend the existing chaincode:

| Beckn Function | Purpose | EPCIS Mapping | Beckn API |
|----------------|---------|--------------|-----------|
| `declareLot()` | Register new commodity lot | ObjectEvent (ADD) | Actor-initiated |
| `getLotsByFilter()` | Query lots with filters | QueryEvent | on_search |
| `transferCustody()` | Transfer custody between actors | TransactionEvent (XFER) | on_confirm |
| `dispatchLot()` | Record physical dispatch | ObjectEvent (DEPART) | on_status |
| `receiveLot()` | Record physical receipt | ObjectEvent (ARRIVE) | on_status |
| `transformLot()` | Processing/transformation | TransformationEvent | on_update |
| `splitMergeLot()` | Split/merge lots | AggregationEvent | on_update |
| `getLotWithLineage()` | Get lot with lineage | - | - |

### Integration Points

```
Existing Function          Beckn Function          Relationship
─────────────────────────────────────────────────────────────────
CreateActor()             declareLot()            Uses CreateActor() to verify actor
GetAsset()                getLotsByFilter()       Queries assets with Beckn filters
CreateTraceabilityEvent() transferCustody()       Creates custody transfer event
CreateTraceabilityEvent() dispatchLot()          Creates dispatch event
CreateTraceabilityEvent() receiveLot()           Creates receipt event
CreateTransformation()    transformLot()          Similar but Beckn-specific
GetAssetLineage()         getLotWithLineage()      Enhanced with Beckn fields
```

---

## Status Flow

The lot status follows this lifecycle:

```
Declared → Confirmed → DispatchPending → Dispatched → InTransit → Received → Processed
```

### Status Meanings

| Status | Description | Ownership | Visible in Search |
|--------|-------------|-----------|-------------------|
| `Declared` | Lot created by farmer, available for purchase | Farmer | ✅ Yes |
| `Confirmed` | Buyer confirmed purchase, awaiting dispatch | Farmer → Buyer (pending) | ❌ No |
| `DispatchPending` | Dispatch initiated, awaiting confirmation (can be cancelled) | Farmer → Buyer (pending) | ❌ No |
| `Dispatched` | Dispatch confirmed, ownership transferred to buyer | Buyer | ❌ No |
| `InTransit` | Lot is being transported | Buyer | ❌ No |
| `Received` | Lot received at destination | Buyer → Consignee | ❌ No |
| `Processed` | Lot transformed | Consignee | ❌ No |

### Dispatch Cancellation

- If cancelled in `DispatchPending` → status reverts to `Confirmed`, ownership unchanged
- If cancelled in `Dispatched` → not allowed (ownership already transferred)

---

## Integration Flow

### Phase 1: Setup - Create Foundation Data

Before using Beckn functions, you need to create base data:

1. **Create Actors** - Register all supply chain participants
2. **Create Facilities** - Register processing facilities
3. **Declare Lots** - Register initial commodity lots

### Phase 2: Discovery - Search for Lots

Buyer searches for available lots using Beckn search API.

**Flow:**
```
BAP /search → BPP /on_search → getLotsByFilter() → Returns matching lots
```

### Phase 3: Selection & Confirmation

Buyer selects a lot and confirms the transfer.

**Flow:**
```
BAP /select → BPP /on_select (mock)
BAP /init → BPP /on_init (mock)
BAP /confirm → BPP /on_confirm → transferCustody() → Updates custody chain
```

### Phase 4: Physical Movement

Buyer initiates dispatch, confirms dispatch, and consignee receives the lot.

**Flow:**
```
BPP /on_status (dispatch_initiated) → initiateDispatch() → Creates pending dispatch
BPP /on_status (dispatch_confirmed) → confirmDispatch() → Transfers ownership to buyer
BPP /on_status (dispatch_cancelled) → cancelDispatch() → Reverts to Confirmed (optional)
BPP /on_status (delivered) → receiveLot() → Records receipt
```

### Phase 5: Processing

Lot undergoes processing/transformation.

**Flow:**
```
BPP /on_update (transform) → transformLot() → Records transformation
BPP /on_update (split_merge) → splitMergeLot() → Records split/merge
```

---

## Complete End-to-End Example

### Scenario: Cotton Lot from Farm to Ginning Facility

**Actors:**
- Farmer (did:example:farmer1)
- Buyer (did:example:buyer1)
- Transporter (did:example:transporter1)
- Consignee (did:example:consignee1)

**Facility:**
- Ginning Facility (FACILITY_001)

**Lot:**
- Cotton Lot (SGTIN-001)

---

## Curl Commands by Flow

### Phase 1: Setup - Create Foundation Data

#### 1.1 Create Farmer Actor

```bash
curl -X POST http://localhost:3000/query/invoke \
  -H "Content-Type: application/json" \
  -d '{
    "functionName": "CreateActor",
    "args": [
      "did:example:farmer1",
      "farmer",
      "John Farmer",
      "REG-001",
      "{\"phone\": \"+91-9876543210\", \"email\": \"john@farm.com\"}",
      "[\"CERT-001\", \"CERT-002\"]",
      "{}"
    ]
  }'
```

**Response:**
```json
{
  "docType": "actor",
  "actor_id": "did:example:farmer1",
  "actor_type": "farmer",
  "name": "John Farmer",
  "status": "Active"
}
```

#### 1.2 Create Buyer Actor

```bash
curl -X POST http://localhost:3000/query/invoke \
  -H "Content-Type: application/json" \
  -d '{
    "functionName": "CreateActor",
    "args": [
      "did:example:buyer1",
      "buyer",
      "Acme Textiles",
      "REG-002",
      "{\"phone\": \"+91-9876543211\", \"email\": \"acme@textiles.com\"}",
      "[\"CERT-003\"]",
      "{}"
    ]
  }'
```

#### 1.3 Create Transporter Actor

```bash
curl -X POST http://localhost:3000/query/invoke \
  -H "Content-Type: application/json" \
  -d '{
    "functionName": "CreateActor",
    "args": [
      "did:example:transporter1",
      "transporter",
      "Fast Logistics",
      "REG-003",
      "{\"phone\": \"+91-9876543212\", \"email\": \"fast@logistics.com\"}",
      "[]",
      "{}"
    ]
  }'
```

#### 1.4 Create Consignee Actor

```bash
curl -X POST http://localhost:3000/query/invoke \
  -H "Content-Type: application/json" \
  -d '{
    "functionName": "CreateActor",
    "args": [
      "did:example:consignee1",
      "consignee",
      "Warehouse Manager",
      "REG-004",
      "{\"phone\": \"+91-9876543213\", \"email\": \"warehouse@acme.com\"}",
      "[]",
      "{}"
    ]
  }'
```

#### 1.5 Create Ginning Facility

```bash
curl -X POST http://localhost:3000/query/invoke \
  -H "Content-Type: application/json" \
  -d '{
    "functionName": "CreateFacility",
    "args": [
      "FACILITY_001",
      "ginning",
      "did:example:consignee1",
      "{\"lat\": 12.9716, \"lng\": 77.5946, \"address\": \"Bangalore\"}",
      "FAC-REG-001",
      "1000",
      "[\"CERT-004\"]",
      "{}"
    ]
  }'
```

#### 1.6 Declare Cotton Lot (Beckn Function)

```bash
curl -X POST http://localhost:3000/query/invoke \
  -H "Content-Type: application/json" \
  -d '{
    "functionName": "declareLot",
    "args": [
      "061414112345678901",  // GS1 SGTIN format
      "did:example:farmer1",
      "cotton",
      "agriculture",
      "GLN-0614141000000",    // GS1 GLN format
      "2024-01-15T00:00:00Z",
      "hash123",
      "5201.00"               // HS Code for raw cotton
    ]
  }'
```

**Response:**
```json
{
  "docType": "lot",
  "lot_id": "SGTIN-001",
  "actor_did": "did:example:farmer1",
  "commodity_type": "cotton",
  "commodity_sector": "agriculture",
  "origin_gln": "GLN-001",
  "harvest_date": "2024-01-15T00:00:00Z",
  "kde_hash": "hash123",
  "status": "Declared",
  "custody_chain": ["did:example:farmer1"]
}
```

---

### Phase 2: Discovery - Search for Lots

#### 2.1 Buyer searches for cotton lots (Beckn Search)

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
        "commoditySector": "agriculture"
      }
    }
  }'
```

**Flow:**
```
BAP /search → BPP /on_search → getLotsByFilter() → Returns lots
```

**Response:**
```json
{
  "context": {
    "action": "on_search",
    "timestamp": "2024-01-15T10:00:00Z"
  },
  "message": {
    "catalog": {
      "items": [
        {
          "id": "SGTIN-001",
          "descriptor": {
            "name": "cotton"
          },
          "@ondc/org/traceability": {
            "lot_id": "SGTIN-001",
            "actor_did": "did:example:farmer1",
            "status": "Declared"
          }
        }
      ]
    }
  }
}
```

---

### Phase 3: Selection & Confirmation

#### 3.1 Buyer selects the lot

```bash
curl -X POST http://localhost:3001/select \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "action": "select",
      "timestamp": "2024-01-15T10:05:00Z"
    },
    "message": {
      "order": {
        "items": [{"id": "SGTIN-001"}],
        "quote": {
          "price": {
            "currency": "INR",
            "value": "10000"
          },
          "title": "Commodity Price"
        }
      }
    }
  }'
```

**Response:**
```json
{
  "context": {"action": "on_select"},
  "message": {
    "order": {
      "items": [{"id": "SGTIN-001"}],
      "quote": {
        "price": {"currency": "INR", "value": "10000"}
      }
    }
  }
}
```

#### 3.2 Buyer initializes order

```bash
curl -X POST http://localhost:3001/init \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "action": "init",
      "timestamp": "2024-01-15T10:10:00Z"
    },
    "message": {
      "order": {
        "items": [{"id": "SGTIN-001"}]
      }
    }
  }'
```

**Response:**
```json
{
  "context": {"action": "on_init"},
  "message": {
    "order": {
      "id": "ORDER_001",
      "state": "PENDING"
    }
  }
}
```

#### 3.3 Buyer confirms custody transfer (Beckn Confirm)

```bash
curl -X POST http://localhost:3001/confirm \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "action": "confirm",
      "timestamp": "2024-01-15T10:15:00Z"
    },
    "message": {
      "order": {
        "id": "ORDER_001",
        "items": [{"id": "SGTIN-001"}],
        "provider": {"id": "did:example:farmer1"},
        "consumer": {"id": "did:example:buyer1"},
        "quote": {"id": "QUOTE_001"}
      }
    }
  }'
```

**Flow:**
```
BAP /confirm → BPP /on_confirm → transferCustody() → Updates custody chain
```

**Response:**
```json
{
  "context": {"action": "on_confirm"},
  "message": {
    "order": {
      "state": "CONFIRMED",
      "@ondc/org/traceability": {
        "transfer_id": "TRANSFER_SGTIN-001_1705334100000",
        "lot_id": "SGTIN-001",
        "from_did": "did:example:farmer1",
        "to_did": "did:example:buyer1",
        "timestamp": "2024-01-15T10:15:00Z"
      }
    }
  }
}
```

**Chaincode State Change:**
- Lot custody chain: `[did:example:farmer1, did:example:buyer1]`
- Lot status: `InTransit`
- New custody transfer record created on ledger

---

### Phase 4: Physical Movement

#### 4.1 Buyer initiates dispatch (Beckn Dispatch Initiated)

```bash
curl -X POST http://localhost:3002/on_status \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "action": "on_status",
      "timestamp": "2024-01-15T11:00:00Z"
    },
    "message": {
      "order": {
        "items": [{"id": "061414112345678901"}],
        "state": "dispatch_initiated",
        "gps_coords": "12.9716,77.5946",
        "transporter_id": "did:example:transporter1",
        "vehicle_id": "VEH001"
      }
    }
  }'
```

**Flow:**
```
BPP /on_status → dispatchLot() → Records dispatch event
```

**Response:**
```json
{
  "context": {"action": "on_status"},
  "message": {
    "order": {
      "updated_at": "2024-01-15T11:00:00Z",
      "@ondc/org/traceability": {
        "event_id": "DISPATCH_SGTIN-001_1705334400000",
        "status": "InTransit"
      }
    }
  }
}
```

**Chaincode State Change:**
- Lot status: `Dispatched`
- Lot current location: `12.9716,77.5946`
- Lot transporter: `did:example:transporter1`
- New dispatch record created on ledger

#### 4.2 Consignee receives the lot (Beckn Receipt)

```bash
curl -X POST http://localhost:3002/on_status \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "action": "on_status",
      "timestamp": "2024-01-15T14:00:00Z"
    },
    "message": {
      "order": {
        "items": [{"id": "SGTIN-001"}],
        "state": "Delivered",
        "consignee_id": "did:example:consignee1",
        "condition_hash": "hash456"
      }
    }
  }'
```

**Flow:**
```
BPP /on_status → receiveLot() → Records receipt event
```

**Response:**
```json
{
  "context": {"action": "on_status"},
  "message": {
    "order": {
      "updated_at": "2024-01-15T14:00:00Z",
      "@ondc/org/traceability": {
        "event_id": "RECEIPT_SGTIN-001_1705334400000",
        "status": "Delivered"
      }
    }
  }
}
```

**Chaincode State Change:**
- Lot status: `Received`
- Lot current custodian: `did:example:consignee1`
- New receipt record created on ledger

---

### Phase 5: Processing

#### 5.1 Create output lot for ginned cotton

First, create the output lot that will be produced:

```bash
curl -X POST http://localhost:3000/query/invoke \
  -H "Content-Type: application/json" \
  -d '{
    "functionName": "declareLot",
    "args": [
      "SGTIN-002",
      "did:example:consignee1",
      "ginned_cotton",
      "agriculture",
      "GLN-002",
      "2024-01-15T15:00:00Z",
      "hash789"
    ]
  }'
```

#### 5.2 Record transformation (Beckn Transform)

```bash
curl -X POST http://localhost:3002/on_update \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "action": "on_update",
      "timestamp": "2024-01-15T15:00:00Z"
    },
    "message": {
      "order": {
        "update_type": "transform",
        "items": [{"id": "SGTIN-001"}],
        "processId": "PROCESS_001",
        "processType": "ginning",
        "inputLotIDs": ["SGTIN-001"],
        "outputLotIDs": ["SGTIN-002"],
        "facilityId": "FACILITY_001",
        "yieldRatio": "0.4"
      }
    }
  }'
```

**Flow:**
```
BPP /on_update → transformLot() → Records transformation event
```

**Response:**
```json
{
  "context": {"action": "on_update"},
  "message": {
    "order": {
      "updated_at": "2024-01-15T15:00:00Z",
      "@ondc/org/traceability": {
        "event_id": "PROCESS_001",
        "update_type": "transform"
      }
    }
  }
}
```

**Chaincode State Change:**
- SGTIN-001 child_lots: `["SGTIN-002"]`
- SGTIN-002 parent_lots: `["SGTIN-001"]`
- SGTIN-002 process_id: `PROCESS_001`
- New transformation record created on ledger

#### 5.3 Verify lot with lineage

```bash
curl -X POST http://localhost:3000/query/custom \
  -H "Content-Type: application/json" \
  -d '{
    "functionName": "getLotWithLineage",
    "args": ["SGTIN-002"]
  }'
```

**Response:**
```json
{
  "lot_id": "SGTIN-002",
  "parent_lots": ["SGTIN-001"],
  "process_id": "PROCESS_001",
  "parent_lot_details": [
    {
      "lot_id": "SGTIN-001",
      "commodity_type": "cotton",
      "actor_did": "did:example:farmer1"
    }
  ]
}
```

---

## Summary of Chaincode State Changes

### Lot SGTIN-001 Lifecycle

| Stage | Status | Custody Chain | Events |
|-------|--------|---------------|--------|
| Declared | `Declared` | `[did:example:farmer1]` | declareLot() |
| Confirmed | `InTransit` | `[did:example:farmer1, did:example:buyer1]` | transferCustody() |
| Dispatched | `Dispatched` | `[did:example:farmer1, did:example:buyer1]` | dispatchLot() |
| Received | `Received` | `[did:example:farmer1, did:example:buyer1]` | receiveLot() |
| Transformed | `Processed` | `[did:example:farmer1, did:example:buyer1]` | transformLot() |

### Ledger Records Created

1. **Actor Records:** 4 actors (farmer, buyer, transporter, consignee)
2. **Facility Record:** 1 ginning facility
3. **Lot Records:** 2 lots (SGTIN-001, SGTIN-002)
4. **Custody Transfer:** 1 transfer record
5. **Dispatch Record:** 1 dispatch record
6. **Receipt Record:** 1 receipt record
7. **Transformation Record:** 1 transformation record

---

## Quick Reference: All Beckn Functions

### Query Functions (Read-Only)

| Function | Endpoint | Purpose |
|----------|----------|---------|
| `getLotsByFilter()` | `/query/custom` | Query lots with filters |
| `getLotWithLineage()` | `/query/custom` | Get lot with parent-child lineage |

### Transaction Functions (Write)

| Function | Endpoint | Purpose |
|----------|----------|---------|
| `declareLot()` | `/query/invoke` | Register new lot |
| `transferCustody()` | `/query/invoke` | Transfer custody |
| `dispatchLot()` | `/query/invoke` | Record dispatch |
| `receiveLot()` | `/query/invoke` | Record receipt |
| `transformLot()` | `/query/invoke` | Record transformation |
| `splitMergeLot()` | `/query/invoke` | Split/merge lots |

---

## Troubleshooting

### Common Errors

**Error:** `Lot SGTIN-001 does not exist`
- **Cause:** Lot not declared before use
- **Fix:** Run `declareLot()` first

**Error:** `Actor did:example:farmer1 does not exist`
- **Cause:** Actor not created before use
- **Fix:** Run `CreateActor()` first

**Error:** `Actor did:example:farmer1 is not the current custodian`
- **Cause:** Trying to transfer from wrong actor
- **Fix:** Ensure `fromDID` matches last entry in custody_chain

**Error:** `Facility FACILITY_001 does not exist`
- **Cause:** Facility not created before transformation
- **Fix:** Run `CreateFacility()` first

### Verify Ledger State

Check what's on the ledger:

```bash
# Query all lots
curl -X POST http://localhost:3000/query/custom \
  -H "Content-Type: application/json" \
  -d '{
    "functionName": "getLotsByFilter",
    "args": ["", "", "", ""]
  }'

# Query specific lot
curl -X POST http://localhost:3000/query/custom \
  -H "Content-Type: application/json" \
  -d '{
    "functionName": "getLotWithLineage",
    "args": ["SGTIN-001"]
  }'

# Query asset history
curl -X GET http://localhost:3000/query/asset/SGTIN-001/history
```

---

## Next Steps

After completing this flow:

1. **Monitor Events:** Use Fabric CLI to monitor chaincode events
2. **Query Ledger:** Verify all records are correctly stored
3. **Test Split/Merge:** Try the `splitMergeLot()` function
4. **Build UI:** Create a frontend to visualize the traceability data
5. **Add More Actors:** Scale to multi-organization scenarios
