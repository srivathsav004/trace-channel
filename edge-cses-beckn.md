# OATS + Beckn Protocol Edge Cases

This document lists all edge cases and validation rules in the system.

---

## Duplicate ID Errors

### 1. Duplicate Lot ID
- **Error:** `Lot {lotId} already exists`
- **Cause:** Attempting to declare a lot with an ID that already exists on the ledger
- **Fix:** Use a unique lot ID (GS1 SGTIN format, 18 digits)
- **Example:** If lot `061414112345678901` exists, cannot declare it again

### 2. Duplicate Actor ID
- **Error:** `Actor {actorId} already exists`
- **Cause:** Attempting to create an actor with an ID that already exists
- **Fix:** Use a unique actor ID (format: `TRST01-{TYPE}-{NUMBER}`)
- **Example:** If `TRST01-FARMER-001` exists, cannot create it again

### 3. Duplicate Facility ID
- **Error:** `Facility {facilityId} already exists`
- **Cause:** Attempting to create a facility with an ID that already exists
- **Fix:** Use a unique facility ID
- **Example:** If `TRST01-GINNING-001` exists, cannot create it again

### 4. Duplicate Process ID
- **Error:** `Process {processId} already exists`
- **Cause:** Attempting to use a process ID that already exists in a transformation
- **Fix:** Use a unique process ID (format: `PROCESS_{timestamp}`)
- **Example:** If `PROCESS_001` exists, use `PROCESS_002` or timestamp-based ID

### 5. Duplicate Dispatch ID
- **Error:** `Dispatch {dispatchId} already exists`
- **Cause:** Attempting to use a dispatch ID that already exists
- **Fix:** Use a unique dispatch ID (format: `DISPATCH_{lotId}_{timestamp}`)
- **Example:** If `DISPATCH_061414112345678901_1705334400000` exists, use a new timestamp

---

## Entity Not Found Errors

### 6. Actor Not Found
- **Error:** `Actor {actorId} does not exist`
- **Cause:** Referencing an actor that hasn't been created
- **Fix:** Create the actor first using `/actors/create`
- **Occurs in:** declare-lot, transfer-custody, receive-lot

### 7. Facility Not Found
- **Error:** `Facility {facilityId} does not exist`
- **Cause:** Referencing a facility that hasn't been created
- **Fix:** Create the facility first using `/facilities/create`
- **Occurs in:** transform-lot

### 8. Lot Not Found
- **Error:** `Lot {lotId} does not exist`
- **Cause:** Referencing a lot that hasn't been declared
- **Fix:** Declare the lot first using `/query/beckn/declare-lot`
- **Occurs in:** transfer-custody, initiate-dispatch, confirm-dispatch, cancel-dispatch, receive-lot, transform-lot

### 9. Input Lot Not Found
- **Error:** `Input lot {lotId} does not exist`
- **Cause:** Input lot for transformation doesn't exist
- **Fix:** Declare the input lot before transformation
- **Occurs in:** transform-lot

### 10. Output Lot Not Found
- **Error:** `Output lot {lotId} does not exist`
- **Cause:** Output lot for transformation hasn't been declared
- **Fix:** Declare output lots before transformation (BPP does this automatically with 3s delay)
- **Occurs in:** transform-lot

---

## Permission & Custody Errors

### 11. Not Current Custodian
- **Error:** `Actor {actorId} is not the current custodian`
- **Cause:** Attempting to transfer custody from an actor who doesn't own the lot
- **Fix:** Ensure `fromDID` matches the last entry in the lot's custody chain
- **Example:** If lot is owned by `TRST01-FARMER-001`, `TRST01-BUYER-001` cannot initiate transfer

### 12. Transfer to Self
- **Error:** `Cannot transfer custody to self`
- **Cause:** Attempting to transfer custody from an actor to themselves
- **Fix:** Ensure `fromDID` and `toDID` are different
- **Example:** `TRST01-FARMER-001` cannot transfer to `TRST01-FARMER-001`

---

## Status Flow Errors

### 13. Wrong Status for Transfer
- **Error:** `Lot status must be 'Declared' for transfer`
- **Cause:** Attempting to transfer custody on a lot that's not in the correct status
- **Fix:** Ensure lot is in `Declared` status before transfer
- **Example:** Cannot transfer a lot that's already `Confirmed` or `Processed`

### 14. Wrong Status for Dispatch
- **Error:** `Lot status must be 'Confirmed' to initiate dispatch`
- **Cause:** Attempting to initiate dispatch on a lot that's not confirmed
- **Fix:** Ensure lot is in `Confirmed` status before dispatch
- **Example:** Cannot dispatch a lot that's still `Declared` or `Received`

### 15. Wrong Status for Receipt
- **Error:** `Lot status must be 'Dispatched' to receive`
- **Cause:** Attempting to receive a lot that hasn't been dispatched
- **Fix:** Ensure lot is in `Dispatched` status before receipt
- **Example:** Cannot receive a lot that's still `Confirmed` or `DispatchPending`

### 16. Wrong Status for Transformation
- **Error:** `Lot status must be 'Received' to transform`
- **Cause:** Attempting to transform a lot that hasn't been received
- **Fix:** Ensure lot is in `Received` status before transformation
- **Example:** Cannot transform a lot that's still `Declared` or `Dispatched`

### 17. Cannot Cancel Confirmed Dispatch
- **Error:** `Cannot cancel dispatch in 'Dispatched' status`
- **Cause:** Attempting to cancel a dispatch that has already been confirmed
- **Fix:** Dispatch can only be cancelled in `DispatchPending` status
- **Example:** Once `confirmDispatch()` is called, dispatch cannot be cancelled

---

## Validation Errors

### 18. Missing Required Fields
- **Error:** `Missing required fields: {fieldList}`
- **Cause:** Required fields not provided in request
- **Fix:** Provide all required fields
- **Examples:**
  - `declare-lot`: lotId, actorDID, commodityType, commoditySector
  - `transfer-custody`: lotId, fromDID, toDID
  - `transform-lot`: processId, inputLotIDs, outputLotIDs

### 19. Empty Output Lots
- **Error:** `outputLots array cannot be empty`
- **Cause:** Transformation request has no output lots defined
- **Fix:** Provide at least one output lot in the transformation request
- **Occurs in:** transform-lot

### 20. Missing Owner Actor ID
- **Error:** `ownerActorId is required for output lot {lotId}`
- **Cause:** Output lot in transformation doesn't specify an owner
- **Fix:** Provide `ownerActorId` for each output lot
- **Occurs in:** transform-lot (dynamic output lot creation)

### 21. Missing Commodity Type
- **Error:** `commodityType is required for output lot {lotId}`
- **Cause:** Output lot in transformation doesn't specify a commodity type
- **Fix:** Provide `commodityType` for each output lot
- **Occurs in:** transform-lot (dynamic output lot creation)

---

## Format Errors

### 22. Invalid Lot ID Format
- **Error:** `Invalid lot ID format`
- **Cause:** Lot ID doesn't follow GS1 SGTIN format (18 digits)
- **Fix:** Use GS1 SGTIN format (18 digits)
- **Example:** Valid: `061414112345678901`, Invalid: `LOT-001`

### 23. Invalid Actor ID Format
- **Error:** `Invalid actor ID format`
- **Cause:** Actor ID doesn't follow the expected format
- **Fix:** Use format `TRST01-{TYPE}-{NUMBER}`
- **Example:** Valid: `TRST01-FARMER-001`, Invalid: `farmer1`

### 24. Invalid Timestamp Format
- **Error:** `Invalid timestamp format`
- **Cause:** Timestamp not in ISO 8601 format
- **Fix:** Use ISO 8601 format
- **Example:** Valid: `2024-01-15T10:00:00Z`, Invalid: `15-01-2024`

### 25. Invalid HS Code Format
- **Error:** `Invalid HS code format`
- **Cause:** HS code doesn't follow the expected format
- **Fix:** Use HS code format (e.g., `5201.00`)
- **Example:** Valid: `5201.00`, Invalid: `5201`

### 26. Invalid Yield Ratio
- **Error:** `Invalid yield ratio`
- **Cause:** Yield ratio is not a valid number or out of range
- **Fix:** Use a decimal between 0 and 1 (e.g., `0.4`)
- **Example:** Valid: `0.4`, Invalid: `1.5` (greater than 1)

---

## Ledger Commit Timing Issues

### 27. Output Lots Not Committed Before Transformation
- **Error:** `Output lot {lotId} does not exist`
- **Cause:** Output lots were declared but not yet committed to ledger when transformLot is called
- **Fix:** System adds 3-second delay after declaring output lots before transformation
- **Note:** This is handled automatically in the BPP

---

## Empty Results

### 28. No Lots Found
- **Error:** No error, but empty result
- **Cause:** Search query returns no matching lots
- **Fix:** Verify lots exist and match the filter criteria
- **Occurs in:** getLotsByFilter, on_search

### 29. No Lineage Found
- **Error:** No error, but empty lineage
- **Cause:** Lot has no parent or child lots
- **Fix:** This is normal for initial lots
- **Occurs in:** getLotWithLineage

---

## Chaincode-Specific Errors

### 30. Endorsement Failure
- **Error:** `endorsement failure during invoke`
- **Cause:** Chaincode transaction failed due to validation or state inconsistency
- **Fix:** Check the specific error message in the response
- **Common causes:** Duplicate IDs, missing entities, wrong status

### 31. Transaction Not Committed
- **Error:** `Transaction not committed`
- **Cause:** Transaction submitted but not yet committed to ledger
- **Fix:** Wait for transaction commit or check transaction status
- **Note:** In Fabric, there's a delay between submission and commit

---

## Summary Table

| Error Type | Common Cause | Fix |
|------------|--------------|-----|
| Duplicate ID | Reusing existing ID | Use unique ID |
| Entity Not Found | Referencing non-existent entity | Create entity first |
| Wrong Custodian | Actor doesn't own lot | Use correct owner |
| Wrong Status | Operation not allowed in current status | Complete prerequisite steps |
| Missing Fields | Required field not provided | Provide all required fields |
| Invalid Format | Data doesn't match expected format | Use correct format |
| Timing Issue | Transaction not committed | Add delay or retry |
