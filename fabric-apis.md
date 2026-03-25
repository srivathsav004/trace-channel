# Product Trace Backend API Documentation

A RESTful API for managing product traceability on Hyperledger Fabric blockchain network.

## Table of Contents

- [Overview](#overview)
- [Base URL](#base-url)
- [Authentication](#authentication)
- [API Endpoints](#api-endpoints)
  - [Health Check](#health-check)
  - [Product Management](#product-management)
  - [Shipping Operations](#shipping-operations)
  - [Receiving Operations](#receiving-operations)
- [Response Format](#response-format)
- [Error Handling](#error-handling)
- [Examples](#examples)

## Overview

This API provides endpoints for creating, tracking, and managing products throughout the supply chain using Hyperledger Fabric blockchain. Each operation is recorded immutably on the blockchain, ensuring complete traceability.

## Base URL

```
http://localhost:3000
```

## Authentication

Currently, no authentication is required. The API uses the ManufacturerMSP identity for all blockchain operations.

## API Endpoints

### Health Check

#### GET /health

Check if the API server is running properly.

**Response:**
```json
{
  "status": "ok"
}
```

---

### Product Management

#### POST /products/create

Create a new product on the blockchain.

**Request Body:**
```json
{
  "id": "string",
  "name": "string", 
  "manufacturer": "string"
}
```

**Parameters:**
- `id` (required): Unique product identifier (SKU)
- `name` (required): Product name
- `manufacturer` (required): Manufacturer name

**Response:**
```json
{
  "success": true,
  "message": "Product created successfully",
  "data": {
    "docType": "product",
    "ID": "SKU-001",
    "Name": "Laptop",
    "Manufacturer": "TechCorp",
    "Owner": "TechCorp",
    "Status": "CREATED",
    "Timestamp": "2026-03-25T07:30:59.614Z"
  }
}
```

---

#### GET /products/get/:id

Retrieve product details by ID.

**Parameters:**
- `id` (path parameter): Product ID

**Response:**
```json
{
  "result": {
    "docType": "product",
    "ID": "SKU-001",
    "Name": "Laptop",
    "Manufacturer": "TechCorp",
    "Owner": "DistributorXYZ",
    "Status": "IN_TRANSIT",
    "Timestamp": "2026-03-25T07:31:55.115Z"
  }
}
```

---

#### GET /products/history/:id

Get complete transaction history for a product.

**Parameters:**
- `id` (path parameter): Product ID

**Response:**
```json
{
  "result": [
    {
      "TxId": "txid1",
      "Timestamp": "2026-03-25T07:30:59.614Z",
      "IsDelete": false,
      "Value": {
        "docType": "product",
        "ID": "SKU-001",
        "Name": "Laptop",
        "Manufacturer": "TechCorp",
        "Owner": "TechCorp",
        "Status": "CREATED"
      }
    },
    {
      "TxId": "txid2", 
      "Timestamp": "2026-03-25T07:31:55.115Z",
      "IsDelete": false,
      "Value": {
        "docType": "product",
        "ID": "SKU-001",
        "Name": "Laptop",
        "Manufacturer": "TechCorp",
        "Owner": "DistributorXYZ",
        "Status": "IN_TRANSIT"
      }
    }
  ]
}
```

---

### Shipping Operations

#### POST /products/ship/:id

Ship a product to a new owner.

**Parameters:**
- `id` (path parameter): Product ID to ship

**Request Body:**
```json
{
  "newOwner": "string"
}
```

**Parameters:**
- `newOwner` (required): Name or identifier of the new owner

**Response:**
```json
{
  "success": true,
  "message": "Product SKU-001 shipped successfully to DistributorXYZ",
  "data": {
    "docType": "product",
    "ID": "SKU-001",
    "Name": "Laptop",
    "Manufacturer": "TechCorp",
    "Owner": "DistributorXYZ",
    "Status": "IN_TRANSIT",
    "Timestamp": "2026-03-25T07:31:55.115Z"
  }
}
```

---

### Receiving Operations

#### POST /products/receive/:id

Mark a product as received by the current owner.

**Parameters:**
- `id` (path parameter): Product ID to receive

**Request Body:**
Empty (no body required)

**Response:**
```json
{
  "success": true,
  "message": "Product SKU-001 received successfully",
  "data": {
    "docType": "product",
    "ID": "SKU-001",
    "Name": "Laptop",
    "Manufacturer": "TechCorp",
    "Owner": "DistributorXYZ",
    "Status": "RECEIVED",
    "Timestamp": "2026-03-25T07:32:13.646Z"
  }
}
```

---

## Response Format

### Success Response

Most endpoints return a standardized success response:

```json
{
  "success": true,
  "message": "Operation completed successfully",
  "data": {
    // Response data specific to the operation
  }
}
```

### Query Response

Query endpoints (GET operations) return data directly:

```json
{
  "result": {
    // Query result data
  }
}
```

## Error Handling

### 400 Bad Request

```json
{
  "error": "Required field missing: id, name, manufacturer are required"
}
```

### 500 Internal Server Error

```json
{
  "error": "Chaincode invocation failed: Error message"
}
```

## Product Status Flow

Products move through the following status lifecycle:

1. **CREATED** - Product is initially created by manufacturer
2. **IN_TRANSIT** - Product has been shipped to new owner
3. **RECEIVED** - Product has been received by the current owner

## Examples

### Complete Product Lifecycle

```bash
# 1. Create a product
curl -X POST http://localhost:3000/products/create \
  -H "Content-Type: application/json" \
  -d '{
    "id": "SKU-LAPTOP-001",
    "name": "Gaming Laptop",
    "manufacturer": "TechCorp"
  }'

# 2. Ship product to distributor
curl -X POST http://localhost:3000/products/ship/SKU-LAPTOP-001 \
  -H "Content-Type: application/json" \
  -d '{
    "newOwner": "GlobalDistributor"
  }'

# 3. Receive product
curl -X POST http://localhost:3000/products/receive/SKU-LAPTOP-001 \
  -H "Content-Type: application/json"

# 4. Check product status
curl -X GET http://localhost:3000/products/get/SKU-LAPTOP-001

# 5. View complete history
curl -X GET http://localhost:3000/products/history/SKU-LAPTOP-001
```

### Quick Health Check

```bash
curl -X GET http://localhost:3000/health
```

## Project Structure

```
backend/
├── apis/
│   ├── products.js      # Product CRUD operations
│   ├── shipping.js      # Shipping operations
│   ├── receiving.js     # Receiving operations
│   └── health.js        # Health check
├── utils/
│   └── responseHelper.js # Response formatting utilities
├── server.js            # Main server file and Fabric integration
└── package.json         # Dependencies and scripts
```

## Dependencies

- **express**: Web framework
- **child_process**: Fabric CLI integration
- **path**: File path utilities

## Running the Server

```bash
# Install dependencies
npm install

# Start the server
node server.js
```

The server will start on `http://localhost:3000` by default.

## Blockchain Configuration

The API is configured to work with:
- **Channel**: `tracechannel`
- **Chaincode**: `product-trace`
- **Orderer**: `localhost:7050`
- **Endorsers**: Manufacturer and Distributor peers

Make sure the Hyperledger Fabric network is running before starting the API server.
