# Beckn Protocol API Test Curls

This document provides curl commands to test the BAP and BPP APIs before integrating the chaincode functions.

## Prerequisites

1. Start the Fabric backend:
```bash
cd oats-network1/backend
npm start
```

2. Start BAP service:
```bash
cd beckn-protocol/bap
npm install
cp .env.example .env
npm start
```

3. Start BPP service:
```bash
cd beckn-protocol/bpp
npm install
cp .env.example .env
npm start
```

## Health Checks (No Chaincode Required)

### Check BAP Health
```bash
curl http://localhost:3001/health
```

**Expected Response:**
```json
{
  "status": "ok",
  "service": "beckn-bap",
  "timestamp": "2024-01-15T10:00:00.000Z"
}
```

### Check BPP Health
```bash
curl http://localhost:3002/health
```

**Expected Response:**
```json
{
  "status": "ok",
  "service": "beckn-bpp",
  "timestamp": "2024-01-15T10:00:00.000Z"
}
```

### Check Fabric Backend Health
```bash
curl http://localhost:3000/health
```

**Expected Response:**
```json
{
  "status": "ok"
}
```

## BAP API Endpoints (Buyer Side)

### 1. Search (Discovery)
**Protocol Event:** Discovery → EPCIS QueryEvent
**Beckn API:** search
**Chaincode:** getLotsByFilter()

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

**Flow:** BAP → BPP (on_search) → Fabric Backend → Chaincode (getLotsByFilter)

**Note:** This will fail until chaincode is integrated because BPP calls the Fabric backend which calls the chaincode.

### 2. Select
**Protocol Event:** Request → EPCIS TransactionEvent (POSS_XFER)
**Beckn API:** select

```bash
curl -X POST http://localhost:3001/select \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "country": "IND",
      "city": "std:080",
      "action": "select",
      "timestamp": "2024-01-15T10:00:00Z",
      "bap_id": "did:example:bap",
      "bpp_id": "did:example:bpp"
    },
    "message": {
      "order": {
        "items": [{
          "id": "SGTIN-001"
        }]
      }
    }
  }'
```

**Flow:** BAP → BPP (on_select) → Returns quote/order details

**Note:** This will work without chaincode as on_select returns mock data.

### 3. Init
**Protocol Event:** Request → EPCIS TransactionEvent (POSS_XFER)
**Beckn API:** init

```bash
curl -X POST http://localhost:3001/init \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "country": "IND",
      "city": "std:080",
      "action": "init",
      "timestamp": "2024-01-15T10:00:00Z",
      "bap_id": "did:example:bap",
      "bpp_id": "did:example:bpp"
    },
    "message": {
      "order": {
        "items": [{
          "id": "SGTIN-001"
        }]
      }
    }
  }'
```

**Flow:** BAP → BPP (on_init) → Creates PENDING state

**Note:** This will work without chaincode as on_init returns mock data.

### 4. Confirm
**Protocol Event:** Approve → EPCIS TransactionEvent (XFER)
**Beckn API:** confirm
**Chaincode:** transferCustody()

```bash
curl -X POST http://localhost:3001/confirm \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "country": "IND",
      "city": "std:080",
      "action": "confirm",
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

**Flow:** BAP → BPP (on_confirm) → Fabric Backend → Chaincode (transferCustody)

**Note:** This will fail until chaincode is integrated.

## BPP API Endpoints (Provider Side)

### 1. on_search (Discovery Response)
**Protocol Event:** Discovery → EPCIS QueryEvent
**Chaincode:** getLotsByFilter()

```bash
curl -X POST http://localhost:3002/on_search \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "country": "IND",
      "city": "std:080",
      "action": "on_search",
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

**Flow:** BPP → Fabric Backend → Chaincode (getLotsByFilter)

**Note:** This will fail until chaincode is integrated.

### 2. on_select (Select Response)
**Protocol Event:** Request → EPCIS TransactionEvent (POSS_XFER)

```bash
curl -X POST http://localhost:3002/on_select \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "country": "IND",
      "city": "std:080",
      "action": "on_select",
      "timestamp": "2024-01-15T10:00:00Z",
      "bap_id": "did:example:bap",
      "bpp_id": "did:example:bpp"
    },
    "message": {
      "order": {
        "items": [{
          "id": "SGTIN-001"
        }]
      }
    }
  }'
```

**Flow:** Returns mock quote/order details

**Note:** This will work without chaincode as it returns mock data.

### 3. on_init (Init Response)
**Protocol Event:** Request → EPCIS TransactionEvent (POSS_XFER)

```bash
curl -X POST http://localhost:3002/on_init \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "country": "IND",
      "city": "std:080",
      "action": "on_init",
      "timestamp": "2024-01-15T10:00:00Z",
      "bap_id": "did:example:bap",
      "bpp_id": "did:example:bpp"
    },
    "message": {
      "order": {
        "items": [{
          "id": "SGTIN-001"
        }]
      }
    }
  }'
```

**Flow:** Returns initialized order with PENDING state

**Note:** This will work without chaincode as it returns mock data.

### 4. on_confirm (Confirm Response)
**Protocol Event:** Approve → EPCIS TransactionEvent (XFER)
**Chaincode:** transferCustody()

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

**Flow:** BPP → Fabric Backend → Chaincode (transferCustody)

**Note:** This will fail until chaincode is integrated.

### 5. on_status (Status Update - Dispatch/Receipt)
**Protocol Event:** Dispatch/Receipt → EPCIS ObjectEvent (DEPART/ARRIVE)
**Chaincode:** dispatchLot() / receiveLot()

```bash
# Dispatch (shipping)
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

```bash
# Receipt (delivered)
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

**Flow:** BPP → Fabric Backend → Chaincode (dispatchLot/receiveLot)

**Note:** This will fail until chaincode is integrated.

### 6. on_update (Transformation/Split-Merge)
**Protocol Event:** Change of State → EPCIS TransformationEvent/AggregationEvent
**Chaincode:** transformLot() / splitMergeLot()

```bash
# Transformation
curl -X POST http://localhost:3002/on_update \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "country": "IND",
      "city": "std:080",
      "action": "on_update",
      "timestamp": "2024-01-15T10:00:00Z",
      "bap_id": "did:example:bap",
      "bpp_id": "did:example:bpp"
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
        "yieldRatio": "0.4"
      }
    }
  }'
```

```bash
# Split/Merge
curl -X POST http://localhost:3002/on_update \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "domain": "oats:agri-traceability",
      "country": "IND",
      "city": "std:080",
      "action": "on_update",
      "timestamp": "2024-01-15T10:00:00Z",
      "bap_id": "did:example:bap",
      "bpp_id": "did:example:bpp"
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
        }
      }
    }
  }'
```

**Flow:** BPP → Fabric Backend → Chaincode (transformLot/splitMergeLot)

**Note:** This will fail until chaincode is integrated.

## Testing Strategy

### Phase 1: Test Without Chaincode (Current State)
Test these endpoints first - they work without chaincode:

1. ✅ Health checks (all services)
2. ✅ BAP → BPP: select → on_select (mock data)
3. ✅ BAP → BPP: init → on_init (mock data)
4. ✅ Direct BPP: on_select (mock data)
5. ✅ Direct BPP: on_init (mock data)

### Phase 2: Test With Chaincode (After Integration)
After integrating chaincode functions, test:

1. BAP → BPP: search → on_search → getLotsByFilter()
2. BAP → BPP: confirm → on_confirm → transferCustody()
3. Direct BPP: on_search → getLotsByFilter()
4. Direct BPP: on_confirm → transferCustody()
5. Direct BPP: on_status → dispatchLot() / receiveLot()
6. Direct BPP: on_update → transformLot() / splitMergeLot()

### Phase 3: Full Flow Test
Test complete end-to-end flows:

1. **Discovery Flow:** search → on_search → getLotsByFilter()
2. **Custody Transfer Flow:** select → on_select → init → on_init → confirm → on_confirm → transferCustody()
3. **Dispatch Flow:** on_status (InTransit) → dispatchLot()
4. **Receipt Flow:** on_status (Delivered) → receiveLot()
5. **Transformation Flow:** on_update (transform) → transformLot()
6. **Split/Merge Flow:** on_update (split_merge) → splitMergeLot()

## Expected Errors Without Chaincode

When testing endpoints that require chaincode before integration, you'll see errors like:

```
{
  "error": {
    "code": "CHAINCODE_ERROR",
    "message": "Function getLotsByFilter not found"
  }
}
```

This is expected and confirms the API routing is working correctly. Once you integrate the chaincode functions, these errors will be resolved.
