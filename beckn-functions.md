# Chaincode Extensions for Beckn Protocol

This directory contains the Beckn-specific chaincode functions that need to be integrated into the existing OATS traceability chaincode.

## Integration Instructions

### Step 1: Copy Functions to Existing Chaincode

Copy the functions from `beckn-functions.js` into your existing chaincode file:
```
oats-network1/chaincode/oats-traceability-javascript/lib/oatsTraceability.js
```

### Step 2: Add Required Functions

Add the following functions to the `OATSTraceabilityContract` class:

1. `declareLot()` - Genesis event for lot declaration
2. `getLotsByFilter()` - Query lots with filters
3. `transferCustody()` - Custody transfer between actors
4. `dispatchLot()` - Record physical dispatch
5. `receiveLot()` - Record physical receipt
6. `transformLot()` - Processing/transformation events
7. `splitMergeLot()` - Split/merge operations
8. `getLotWithLineage()` - Get lot with parent-child lineage
9. `generateDigitalSignature()` - Helper for digital signatures

### Step 3: Rebuild and Deploy Chaincode

After adding the functions, rebuild and redeploy the chaincode:

```bash
cd oats-network1
./network.sh deployCC -ccn oats-traceability -ccp ../chaincode/oats-traceability-javascript -ccl javascript -ccv 1.1 -ccs 1 -ccinit '{"Args":[]}'
```

### Step 4: Test the New Functions

Test the new chaincode functions using the Fabric CLI or through the Beckn API services.

## Function Mapping

| Beckn Protocol Event | Chaincode Function | EPCIS Event Type |
|---------------------|---------------------|------------------|
| Declare | declareLot() | ObjectEvent (ADD) |
| Discovery | getLotsByFilter() | QueryEvent |
| Approve | transferCustody() | TransactionEvent (XFER) |
| Dispatch | dispatchLot() | ObjectEvent (DEPART) |
| Receipt | receiveLot() | ObjectEvent (ARRIVE) |
| Change of State (Transform) | transformLot() | TransformationEvent |
| Change of State (Split/Merge) | splitMergeLot() | AggregationEvent |

## Data Models

### Lot Model
```json
{
  "docType": "lot",
  "lot_id": "SGTIN-001",
  "actor_did": "did:example:actor1",
  "commodity_type": "cotton",
  "commodity_sector": "agriculture",
  "origin_gln": "GLN-001",
  "harvest_date": "2024-01-15T00:00:00Z",
  "kde_hash": "hash123",
  "created_timestamp": "1705334400",
  "status": "Declared",
  "custody_chain": ["did:example:actor1"],
  "epcis_event_type": "ObjectEvent",
  "epcis_action": "ADD"
}
```

### Custody Transfer Model
```json
{
  "docType": "custody_transfer",
  "transfer_id": "TRANSFER_LOT001_1705334400",
  "lot_id": "LOT001",
  "from_did": "did:example:actor1",
  "to_did": "did:example:actor2",
  "commercial_terms_hash": "hash456",
  "timestamp": "1705334400",
  "status": "Completed",
  "epcis_event_type": "TransactionEvent",
  "epcis_action": "XFER"
}
```

## Notes

- All functions emit Beckn protocol events using `ctx.stub.setEvent()`
- Digital signatures are generated using a helper function (should be enhanced with proper cryptography)
- Lineage tracking is maintained through parent_lot_ids and child_lot_ids arrays
- The functions integrate with existing actor, asset, and facility models
