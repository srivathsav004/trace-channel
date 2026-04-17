# Beckn Protocol Integration for OATS

This directory contains the Beckn protocol layer that sits on top of the Hyperledger Fabric networks (oats-network1 and oats-network2).

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Beckn Protocol Layer (L3)               │
├─────────────────────────────────────────────────────────────┤
│  BAP (Beckn Application Platform)  │  BPP (Beckn Platform)  │
│  - search, select, init, confirm   │  - on_search, on_confirm│
│  - Buyer/Searcher side             │  - on_status, on_update│
│  - Port: 3001                      │  - Provider/Seller side│
│                                     │  - Port: 3002          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Hyperledger Fabric Networks (L1+L2)            │
├─────────────────────────────────────────────────────────────┤
│  oats-network1 (TRST01)  │  oats-network2 (TRST01)         │
│  - Chaincode              │  - Chaincode                   │
│  - Peer endpoints         │  - Peer endpoints              │
│  - oatschannel            │  - oatschannel                 │
└─────────────────────────────────────────────────────────────┘
```

## Protocol Event Mapping

| Protocol Event | EPCIS 2.0 Event | Beckn API Trigger | Chaincode Function |
|----------------|-----------------|-------------------|-------------------|
| Declare | ObjectEvent (ADD) | Actor-initiated | declareLot() |
| Discovery | QueryEvent | search / on_search | getLotsByFilter() |
| Request | TransactionEvent (POSS_XFER) | select / init / confirm | (endorsement request) |
| Approve | TransactionEvent (XFER) | on_confirm | transferCustody() |
| Dispatch | ObjectEvent (DEPART) | on_status (shipping) | dispatchLot() |
| Receipt | ObjectEvent (ARRIVE) | on_status (delivered) | receiveLot() |
| Change of State | TransformationEvent/AggregationEvent | update / on_update | transformLot(), splitMergeLot() |

## Directory Structure

```
beckn-protocol/
├── bap/                    # Beckn Application Platform (buyer side)
│   ├── src/
│   ├── package.json
│   └── .env
├── bpp/                    # Beckn Platform Provider (seller side)
│   ├── src/
│   ├── package.json
│   └── .env
├── chaincode-extensions/   # Extended chaincode functions for Beckn
│   └── beckn-functions.js
├── config/                 # Configuration files
│   ├── beckn-schema.json
│   └── network-config.json
└── README.md
```

## Setup Instructions

### Prerequisites
- Node.js >= 18
- Hyperledger Fabric networks running (oats-network1, oats-network2)
- Existing OATS chaincode deployed

### Installation

1. **Install BAP dependencies:**
   ```bash
   cd bap
   npm install
   ```

2. **Install BPP dependencies:**
   ```bash
   cd bpp
   npm install
   ```

3. **Configure environment variables:**
   - Copy `.env.example` to `.env` in both bap and bpp directories
   - Update with your Fabric network endpoints

4. **Deploy chaincode extensions:**
   - The chaincode extensions need to be merged into the existing chaincode
   - See `chaincode-extensions/README.md` for details

### Running the Services

**Start BAP (Buyer side):**
```bash
cd bap
npm start
```

**Start BPP (Provider side):**
```bash
cd bpp
npm start
```

## API Endpoints

### BAP Endpoints (Port 3001)
- `POST /search` - Search for commodity lots
- `POST /select` - Select a lot for purchase
- `POST /init` - Initialize transaction
- `POST /confirm` - Confirm transaction

### BPP Endpoints (Port 3002)
- `POST /on_search` - Respond to search queries
- `POST /on_confirm` - Respond to confirmation
- `POST /on_status` - Update status (dispatch/receipt)
- `POST /on_update` - Handle transformations

## Development

For local development, you can run both BAP and BPP services on the same machine using different ports.

### Testing the Integration

1. Start both BAP and BPP services
2. Use the provided curl examples in `bap/test-curls.md` and `bpp/test-curls.md`
3. Verify chaincode transactions are recorded on the Fabric ledger

## Network Configuration

The Beckn protocol layer connects to:
- oats-network1: localhost:7051 (peer0.trst01.example.com)
- oats-network2: localhost:8051 (peer0.trst01.example.com)
- Channel: oatschannel
- Chaincode: oats-traceability

## Notes

- The BAP and BPP services communicate via HTTP/HTTPS
- Chaincode calls are made through the existing Fabric CLI or Gateway SDK
- EPCIS 2.0 compliance is maintained through the event mapping
- DID (Decentralized Identifier) is used for actor identity
- GS1 SGTIN-based Lot IDs are assigned during the Declare event
