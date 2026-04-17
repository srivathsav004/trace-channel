# Beckn Protocol Flow Documentation

This document explains the complete Beckn protocol flow for OATS traceability, including parameters, expected returns, and the end-to-end journey through BAP, BPP, and Fabric chaincode.

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
│  - /query/custom - for query functions                      │
│  - /query/invoke - for transaction functions                │
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

## Chaincode Functions Overview

| Function | Protocol Event | EPCIS Event | Beckn API | Purpose |
|----------|----------------|-------------|-----------|---------|
| `declareLot()` | Declare | ObjectEvent (ADD) | Actor-initiated | Register new commodity lot |
| `getLotsByFilter()` | Discovery | QueryEvent | on_search | Query lots with filters |
| `transferCustody()` | Approve | TransactionEvent (XFER) | on_confirm | Transfer custody between actors |
| `dispatchLot()` | Dispatch | ObjectEvent (DEPART) | on_status | Record physical dispatch |
| `receiveLot()` | Receipt | ObjectEvent (ARRIVE) | on_status | Record physical receipt |
| `transformLot()` | Change of State | TransformationEvent | on_update | Processing/transformation |
| `splitMergeLot()` | Change of State | AggregationEvent | on_update | Split/merge operations |
| `getLotWithLineage()` | Query | - | - | Get lot with parent-child lineage |

---

## Flow 1: Discovery (Search)

**Purpose:** Buyer searches for available commodity lots based on criteria.

### Request Flow

```
BAP (search) → BPP (on_search) → Fabric Backend (/query/custom) → Chaincode (getLotsByFilter)
```

### Parameters

#### BAP Request: `POST /search`

```json
{
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
      "complianceStatus": "certified",
      "certificationType": "organic",
      "originZone": "south"
    }
  }
}
```

**Parameters:**
- `commoditySector`: Filter by sector (agriculture, fishery, dairy)
- `complianceStatus`: Filter by compliance status (certified, pending)
- `certificationType`: Filter by certification type (organic, fairtrade)
- `originZone`: Filter by geographic zone

### Chaincode Function: `getLotsByFilter()`

```javascript
async getLotsByFilter(ctx, commoditySector, complianceStatus, certificationType, originZone)
```

**Parameters:**
- `commoditySector` (string, optional): Filter by commodity sector
- `complianceStatus` (string, optional): Filter by compliance status
- `certificationType` (string, optional): Filter by certification type
- `originZone` (string, optional): Filter by origin zone

**Returns:** JSON array of matching lots

```json
[
  {
    "docType": "lot",
    "lot_id": "SGTIN-001",
    "actor_did": "did:example:farmer1",
    "commodity_type": "cotton",
    "commodity_sector": "agriculture",
    "origin_gln": "GLN-001",
    "harvest_date": "2024-01-15T00:00:00Z",
    "kde_hash": "hash123",
    "created_timestamp": "1705334400",
    "status": "Declared",
    "custody_chain": ["did:example:farmer1"]
  }
]
```

### Expected Response

```json
{
  "context": {
    "domain": "oats:agri-traceability",
    "action": "on_search",
    "timestamp": "2024-01-15T10:00:00Z"
  },
  "message": {
    "catalog": {
      "descriptor": {
        "name": "OATS Agricultural Commodities",
        "code": "OATS-AGRI"
      },
      "items": [
        {
          "id": "SGTIN-001",
          "descriptor": {
            "name": "cotton",
            "code": "agriculture"
          },
          "@ondc/org/traceability": {
            "lot_id": "SGTIN-001",
            "actor_did": "did:example:farmer1",
            "origin_gln": "GLN-001",
            "harvest_date": "2024-01-15T00:00:00Z",
            "status": "Declared"
          }
        }
      ]
    }
  }
}
```

---

## Flow 2: Selection (Select)

**Purpose:** Buyer selects a lot and requests a quote.

### Request Flow

```
BAP (select) → BPP (on_select)
```

### Parameters

#### BAP Request: `POST /select`

```json
{
  "context": {
    "domain": "oats:agri-traceability",
    "country": "IND",
    "city": "std:080",
    "action": "select",
    "timestamp": "2024-01-15T10:00:00Z"
  },
  "message": {
    "order": {
      "items": [{
        "id": "SGTIN-001"
      }]
    }
  }
}
```

**Parameters:**
- `items[0].id`: Lot ID being selected

### Expected Response

```json
{
  "context": {
    "action": "on_select"
  },
  "message": {
    "order": {
      "items": [{"id": "SGTIN-001"}],
      "quote": {
        "price": {
          "currency": "INR",
          "value": "10000"
        },
        "breakup": [{
          "title": "Commodity Price",
          "price": {
            "currency": "INR",
            "value": "10000"
          }
        }]
      }
    }
  }
}
```

**Note:** This flow uses mock data and doesn't invoke chaincode.

---

## Flow 3: Initialization (Init)

**Purpose:** Buyer initializes the order, creating a PENDING state.

### Request Flow

```
BAP (init) → BPP (on_init)
```

### Parameters

#### BAP Request: `POST /init`

```json
{
  "context": {
    "domain": "oats:agri-traceability",
    "action": "init",
    "timestamp": "2024-01-15T10:00:00Z"
  },
  "message": {
    "order": {
      "items": [{
        "id": "SGTIN-001"
      }]
    }
  }
}
```

### Expected Response

```json
{
  "context": {
    "action": "on_init"
  },
  "message": {
    "order": {
      "id": "ORDER_001",
      "items": [{"id": "SGTIN-001"}],
      "state": "PENDING",
      "created_at": "2024-01-15T10:00:00Z"
    }
  }
}
```

**Note:** This flow uses mock data and doesn't invoke chaincode.

---

## Flow 4: Custody Transfer (Confirm)

**Purpose:** Buyer confirms the order, triggering custody transfer on the ledger.

### Request Flow

```
BAP (confirm) → BPP (on_confirm) → Fabric Backend (/query/invoke) → Chaincode (transferCustody)
```

### Parameters

#### BAP Request: `POST /confirm`

```json
{
  "context": {
    "domain": "oats:agri-traceability",
    "action": "confirm",
    "timestamp": "2024-01-15T10:00:00Z"
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
}
```

**Parameters:**
- `items[0].id`: Lot ID being transferred
- `provider.id`: DID of current custodian (from)
- `consumer.id`: DID of new custodian (to)
- `quote.id`: Commercial terms hash

### Chaincode Function: `transferCustody()`

```javascript
async transferCustody(ctx, lotID, fromDID, toDID, commercialTermsHash, timestamp)
```

**Parameters:**
- `lotID` (string): Lot ID being transferred
- `fromDID` (string): DID of current custodian
- `toDID` (string): DID of new custodian
- `commercialTermsHash` (string): Hash of commercial terms
- `timestamp` (string): Transfer timestamp (ISO 8601)

**Returns:** Transfer record

```json
{
  "docType": "custody_transfer",
  "transfer_id": "TRANSFER_SGTIN-001_1705334400000",
  "lot_id": "SGTIN-001",
  "from_did": "did:example:farmer1",
  "to_did": "did:example:buyer1",
  "commercial_terms_hash": "QUOTE_001",
  "timestamp": "2024-01-15T10:00:00Z",
  "status": "Completed",
  "epcis_event_type": "TransactionEvent",
  "epcis_action": "XFER",
  "digital_signature": "SIG_tx123_1705334400_TRANSFER_SGTIN-001_1705334400000"
}
```

**Side Effects:**
- Updates lot custody chain to include new custodian
- Changes lot status to "InTransit"
- Creates custody transfer record on ledger
- Emits "TransferCustody" event

### Expected Response

```json
{
  "context": {
    "action": "on_confirm"
  },
  "message": {
    "order": {
      "id": "ORDER_001",
      "state": "CONFIRMED",
      "updated_at": "2024-01-15T10:00:00Z",
      "@ondc/org/traceability": {
        "transfer_id": "TRANSFER_SGTIN-001_1705334400000",
        "lot_id": "SGTIN-001",
        "from_did": "did:example:farmer1",
        "to_did": "did:example:buyer1",
        "timestamp": "2024-01-15T10:00:00Z"
      }
    }
  }
}
```

---

## Flow 5: Dispatch (Status Update)

**Purpose:** Provider dispatches the lot, recording physical departure.

### Request Flow

```
BPP (on_status) → Fabric Backend (/query/invoke) → Chaincode (dispatchLot)
```

### Parameters

#### BPP Request: `POST /on_status` (Dispatch)

```json
{
  "context": {
    "domain": "oats:agri-traceability",
    "action": "on_status",
    "timestamp": "2024-01-15T10:00:00Z"
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
}
```

**Parameters:**
- `items[0].id`: Lot ID being dispatched
- `state`: Must be "InTransit" or "Shipped"
- `gps_coords`: GPS coordinates (lat,lng format)
- `transporter_id`: DID of transporter
- `vehicle_id`: Vehicle registration ID

### Chaincode Function: `dispatchLot()`

```javascript
async dispatchLot(ctx, lotID, dispatchTimestamp, gpsCoords, transporterDID, vehicleID)
```

**Parameters:**
- `lotID` (string): Lot ID being dispatched
- `dispatchTimestamp` (string): Dispatch timestamp (ISO 8601)
- `gpsCoords` (string): GPS coordinates (lat,lng format)
- `transporterDID` (string): DID of transporter
- `vehicleID` (string): Vehicle registration ID

**Returns:** Dispatch record

```json
{
  "docType": "dispatch",
  "dispatch_id": "DISPATCH_SGTIN-001_1705334400000",
  "lot_id": "SGTIN-001",
  "dispatch_timestamp": "2024-01-15T10:00:00Z",
  "gps_coords": "12.9716,77.5946",
  "transporter_did": "did:example:transporter1",
  "vehicle_id": "VEH001",
  "status": "InTransit",
  "epcis_event_type": "ObjectEvent",
  "epcis_action": "DEPART",
  "read_point": "GLN-001",
  "digital_signature": "SIG_tx123_1705334400_DISPATCH_SGTIN-001_1705334400000"
}
```

**Side Effects:**
- Updates lot status to "Dispatched"
- Sets current location to GPS coordinates
- Records transporter and vehicle information
- Creates dispatch record on ledger
- Emits "DispatchLot" event

### Expected Response

```json
{
  "context": {
    "action": "on_status"
  },
  "message": {
    "order": {
      "updated_at": "2024-01-15T10:00:00Z",
      "@ondc/org/traceability": {
        "lot_id": "SGTIN-001",
        "event_id": "DISPATCH_SGTIN-001_1705334400000",
        "status": "InTransit",
        "timestamp": "2024-01-15T10:00:00Z"
      }
    }
  }
}
```

---

## Flow 6: Receipt (Status Update)

**Purpose:** Consignee receives the lot, recording physical arrival.

### Request Flow

```
BPP (on_status) → Fabric Backend (/query/invoke) → Chaincode (receiveLot)
```

### Parameters

#### BPP Request: `POST /on_status` (Receipt)

```json
{
  "context": {
    "domain": "oats:agri-traceability",
    "action": "on_status",
    "timestamp": "2024-01-15T10:00:00Z"
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
}
```

**Parameters:**
- `items[0].id`: Lot ID being received
- `state`: Must be "Delivered" or "Received"
- `consignee_id`: DID of consignee
- `condition_hash`: Hash of condition/quality data

### Chaincode Function: `receiveLot()`

```javascript
async receiveLot(ctx, lotID, receiptTimestamp, consigneeDID, conditionHash)
```

**Parameters:**
- `lotID` (string): Lot ID being received
- `receiptTimestamp` (string): Receipt timestamp (ISO 8601)
- `consigneeDID` (string): DID of consignee
- `conditionHash` (string): Hash of condition/quality data

**Returns:** Receipt record

```json
{
  "docType": "receipt",
  "receipt_id": "RECEIPT_SGTIN-001_1705334400000",
  "lot_id": "SGTIN-001",
  "receipt_timestamp": "2024-01-15T10:00:00Z",
  "consignee_did": "did:example:consignee1",
  "condition_hash": "hash456",
  "status": "Delivered",
  "epcis_event_type": "ObjectEvent",
  "epcis_action": "ARRIVE",
  "digital_signature": "SIG_tx123_1705334400_RECEIPT_SGTIN-001_1705334400000"
}
```

**Side Effects:**
- Updates lot status to "Received"
- Sets current custodian to consignee
- Records condition hash
- Creates receipt record on ledger
- Emits "ReceiveLot" event

### Expected Response

```json
{
  "context": {
    "action": "on_status"
  },
  "message": {
    "order": {
      "updated_at": "2024-01-15T10:00:00Z",
      "@ondc/org/traceability": {
        "lot_id": "SGTIN-001",
        "event_id": "RECEIPT_SGTIN-001_1705334400000",
        "status": "Delivered",
        "timestamp": "2024-01-15T10:00:00Z"
      }
    }
  }
}
```

---

## Flow 7: Transformation (Update)

**Purpose:** Record processing/transformation of lots (e.g., ginning, milling).

### Request Flow

```
BPP (on_update) → Fabric Backend (/query/invoke) → Chaincode (transformLot)
```

### Parameters

#### BPP Request: `POST /on_update` (Transform)

```json
{
  "context": {
    "domain": "oats:agri-traceability",
    "action": "on_update",
    "timestamp": "2024-01-15T10:00:00Z"
  },
  "message": {
    "order": {
      "update_type": "transform",
      "items": [{
        "id": "SGTIN-001"
      }],
      "processId": "PROCESS_001",
      "processType": "ginning",
      "inputLotIDs": ["SGTIN-001"],
      "outputLotIDs": ["SGTIN-002"],
      "facilityId": "FACILITY_001",
      "yieldRatio": "0.4",
      "metadata": {
        "temperature": "25",
        "humidity": "60"
      }
    }
  }
}
```

**Parameters:**
- `update_type`: Must be "transform"
- `processId`: Unique process ID
- `processType`: Type of transformation (ginning, milling, etc.)
- `inputLotIDs`: JSON array of input lot IDs
- `outputLotIDs`: JSON array of output lot IDs
- `facilityId`: Facility where transformation occurs
- `yieldRatio`: Yield ratio (output/input)
- `metadata`: Additional process metadata

### Chaincode Function: `transformLot()`

```javascript
async transformLot(ctx, processId, processType, inputLotIDs, outputLotIDs, facilityId, yieldRatio, metadata)
```

**Parameters:**
- `processId` (string): Unique process ID
- `processType` (string): Type of transformation
- `inputLotIDs` (string): JSON array of input lot IDs
- `outputLotIDs` (string): JSON array of output lot IDs
- `facilityId` (string): Facility ID
- `yieldRatio` (string): Yield ratio as string
- `metadata` (string): JSON string of metadata

**Returns:** Transformation record

```json
{
  "docType": "transformation",
  "process_id": "PROCESS_001",
  "process_type": "ginning",
  "input_lot_ids": ["SGTIN-001"],
  "output_lot_ids": ["SGTIN-002"],
  "facility_id": "FACILITY_001",
  "timestamp": "1705334400",
  "yield_ratio": 0.4,
  "metadata": {
    "temperature": "25",
    "humidity": "60"
  },
  "epcis_event_type": "TransformationEvent",
  "digital_signature": "SIG_tx123_1705334400_PROCESS_001"
}
```

**Side Effects:**
- Creates transformation record on ledger
- Updates parent-child lineage for all lots
- Adds output lots to child_lots of input lots
- Sets parent_lots and process_id on output lots
- Emits "TransformLot" event

### Expected Response

```json
{
  "context": {
    "action": "on_update"
  },
  "message": {
    "order": {
      "updated_at": "2024-01-15T10:00:00Z",
      "@ondc/org/traceability": {
        "event_id": "PROCESS_001",
        "update_type": "transform",
        "timestamp": "2024-01-15T10:00:00Z"
      }
    }
  }
}
```

---

## Flow 8: Split/Merge (Update)

**Purpose:** Split or merge lots for mass-balance traceability.

### Request Flow

```
BPP (on_update) → Fabric Backend (/query/invoke) → Chaincode (splitMergeLot)
```

### Parameters

#### BPP Request: `POST /on_update` (Split)

```json
{
  "context": {
    "domain": "oats:agri-traceability",
    "action": "on_update",
    "timestamp": "2024-01-15T10:00:00Z"
  },
  "message": {
    "order": {
      "update_type": "split_merge",
      "items": [{
        "id": "SGTIN-001"
      }],
      "operationId": "OP_001",
      "operationType": "SPLIT",
      "sourceLotIDs": ["SGTIN-001"],
      "targetLotIDs": ["SGTIN-002", "SGTIN-003"],
      "quantities": {
        "SGTIN-002": "500",
        "SGTIN-003": "500"
      },
      "metadata": {}
    }
  }
}
```

**Parameters:**
- `update_type`: Must be "split_merge"
- `operationId`: Unique operation ID
- `operationType`: "SPLIT" or "MERGE"
- `sourceLotIDs`: JSON array of source lot IDs
- `targetLotIDs`: JSON array of target lot IDs
- `quantities`: JSON object mapping lot IDs to quantities
- `metadata`: Additional operation metadata

### Chaincode Function: `splitMergeLot()`

```javascript
async splitMergeLot(ctx, operationId, operationType, sourceLotIDs, targetLotIDs, quantities, metadata)
```

**Parameters:**
- `operationId` (string): Unique operation ID
- `operationType` (string): "SPLIT" or "MERGE"
- `sourceLotIDs` (string): JSON array of source lot IDs
- `targetLotIDs` (string): JSON array of target lot IDs
- `quantities` (string): JSON object of quantities
- `metadata` (string): JSON string of metadata

**Returns:** Split/Merge operation record

```json
{
  "docType": "split_merge_operation",
  "operation_id": "OP_001",
  "operation_type": "SPLIT",
  "source_lot_ids": ["SGTIN-001"],
  "target_lot_ids": ["SGTIN-002", "SGTIN-003"],
  "quantities": {
    "SGTIN-002": "500",
    "SGTIN-003": "500"
  },
  "timestamp": "1705334400",
  "metadata": {},
  "epcis_event_type": "AggregationEvent",
  "digital_signature": "SIG_tx123_1705334400_OP_001"
}
```

**Side Effects:**
- Creates split/merge operation record on ledger
- Updates lot lineage based on operation type
- For SPLIT: one source → multiple targets
- For MERGE: multiple sources → one target
- Emits "SplitMergeLot" event

### Expected Response

```json
{
  "context": {
    "action": "on_update"
  },
  "message": {
    "order": {
      "updated_at": "2024-01-15T10:00:00Z",
      "@ondc/org/traceability": {
        "event_id": "OP_001",
        "update_type": "split_merge",
        "timestamp": "2024-01-15T10:00:00Z"
      }
    }
  }
}
```

---

## Complete End-to-End Example

### Scenario: Cotton Lot from Farm to Ginning Facility

#### Step 1: Farmer declares lot (Direct chaincode call, not Beckn)

```bash
curl -X POST http://localhost:3000/query/invoke \
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

#### Step 2: Buyer searches for lots (Discovery)

```bash
curl -X POST http://localhost:3001/search \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "action": "search",
      "timestamp": "2024-01-15T10:00:00Z"
    },
    "message": {
      "intent": {
        "commoditySector": "agriculture"
      }
    }
  }'
```

#### Step 3: Buyer selects lot (Selection)

```bash
curl -X POST http://localhost:3001/select \
  -H "Content-Type: application/json" \
  -d '{
    "context": {"action": "select"},
    "message": {
      "order": {"items": [{"id": "SGTIN-001"}]}
    }
  }'
```

#### Step 4: Buyer initializes order (Init)

```bash
curl -X POST http://localhost:3001/init \
  -H "Content-Type: application/json" \
  -d '{
    "context": {"action": "init"},
    "message": {
      "order": {"items": [{"id": "SGTIN-001"}]}
    }
  }'
```

#### Step 5: Buyer confirms transfer (Custody Transfer)

```bash
curl -X POST http://localhost:3001/confirm \
  -H "Content-Type: application/json" \
  -d '{
    "context": {"action": "confirm"},
    "message": {
      "order": {
        "items": [{"id": "SGTIN-001"}],
        "provider": {"id": "did:example:farmer1"},
        "consumer": {"id": "did:example:buyer1"},
        "quote": {"id": "QUOTE_001"}
      }
    }
  }'
```

#### Step 6: Provider dispatches lot (Dispatch)

```bash
curl -X POST http://localhost:3002/on_status \
  -H "Content-Type: application/json" \
  -d '{
    "context": {"action": "on_status"},
    "message": {
      "order": {
        "items": [{"id": "SGTIN-001"}],
        "state": "InTransit",
        "gps_coords": "12.9716,77.5946",
        "transporter_id": "did:example:transporter1",
        "vehicle_id": "VEH001"
      }
    }
  }'
```

#### Step 7: Consignee receives lot (Receipt)

```bash
curl -X POST http://localhost:3002/on_status \
  -H "Content-Type: application/json" \
  -d '{
    "context": {"action": "on_status"},
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

#### Step 8: Ginning facility transforms lot (Transformation)

```bash
curl -X POST http://localhost:3002/on_update \
  -H "Content-Type: application/json" \
  -d '{
    "context": {"action": "on_update"},
    "message": {
      "order": {
        "update_type": "transform",
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

---

## Testing Without Chaincode Integration

All BPP routes now include mock data fallbacks. When chaincode functions are not yet integrated, the BPP will:

1. Try to call the Fabric backend
2. If the chaincode function doesn't exist (returns error), log a message
3. Return mock data matching the expected response format

This allows you to test the complete Beckn protocol flow before integrating the chaincode functions.

**Console Output Example:**
```
Chaincode function getLotsByFilter not found, returning mock data for testing
```

---

## Next Steps

1. **Integrate Chaincode Functions:** Copy functions from `chaincode-extensions/beckn-functions.js` into `oats-network1/chaincode/oats-traceability-javascript/lib/oatsTraceability.js`

2. **Redeploy Chaincode:** Update the chaincode on the Fabric network

3. **Test with Real Chaincode:** Restart BPP service and test flows again - it will now use real chaincode instead of mock data

4. **Verify Ledger Updates:** Use Fabric CLI or backend API to verify that ledger is being updated correctly
