# Electronic Trading API

A RESTful API design for a simplified electronic trading platform, specified using OpenAPI 3.1.

The API allows authenticated clients to:

* View available financial instruments
* Submit market and limit orders
* Retrieve and filter existing orders
* Cancel eligible orders
* View current positions

The complete API contract is defined in [`openapi.yaml`](./openapi.yaml).

## Design Goals

The API was designed around a few core principles:

* **Predictable REST semantics** — resources are represented through clear endpoints and standard HTTP methods.
* **Safe order submission** — order creation supports idempotency to protect clients from accidentally submitting duplicate orders when retrying requests.
* **Consistent error handling** — errors use a shared machine-readable structure with an error code, human-readable message, and request identifier.
* **Explicit validation** — invalid quantities, unsupported order values, and missing limit prices are rejected through schema validation.
* **Scalability** — collection endpoints support cursor-based pagination and the API defines rate-limit behavior.
* **Reusability** — common schemas, parameters, responses, and headers are defined as reusable OpenAPI components.

## API Overview

### Instruments

| Method | Endpoint                | Description                    |
| ------ | ----------------------- | ------------------------------ |
| `GET`  | `/instruments`          | List tradable instruments      |
| `GET`  | `/instruments/{symbol}` | Retrieve a specific instrument |

### Orders

| Method   | Endpoint             | Description               |
| -------- | -------------------- | ------------------------- |
| `POST`   | `/orders`            | Submit a new order        |
| `GET`    | `/orders`            | List and filter orders    |
| `GET`    | `/orders/{order_id}` | Retrieve a specific order |
| `DELETE` | `/orders/{order_id}` | Cancel an eligible order  |

### Positions

| Method | Endpoint              | Description                           |
| ------ | --------------------- | ------------------------------------- |
| `GET`  | `/positions`          | List current positions                |
| `GET`  | `/positions/{symbol}` | Retrieve a position for an instrument |

## Order Submission

The API supports both **market** and **limit** orders.

Example limit order:

```json
{
  "symbol": "AAPL",
  "side": "buy",
  "type": "limit",
  "quantity": 100,
  "limit_price": 220.00
}
```

Example market order:

```json
{
  "symbol": "AAPL",
  "side": "sell",
  "type": "market",
  "quantity": 50
}
```

`quantity` must be a positive integer. A `limit_price` greater than zero is required when the order type is `limit`.

## Idempotency

Creating an order is a state-changing operation, so `POST /orders` requires an `Idempotency-Key` header.

For example:

```http
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

This allows a client to safely retry a request if a connection fails or a response is lost without unintentionally creating the same order twice.

If an idempotency key is reused with a different request, the API returns `409 Conflict`.

## Authentication

The API assumes API-key authentication using the `X-API-Key` request header:

```http
X-API-Key: <api-key>
```

The specification describes the authentication contract only. Key generation, storage, rotation, and revocation would be handled by the implementation in a production system.

## Error Handling

Errors follow a consistent structure:

```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "The request body could not be processed.",
    "request_id": "req_a812cd91"
  }
}
```

The machine-readable `code` allows clients to handle errors programmatically, while `message` provides additional information for developers. `request_id` can be used for tracing and debugging.

The API uses standard HTTP status codes including:

| Status | Meaning                                                     |
| ------ | ----------------------------------------------------------- |
| `400`  | Invalid or malformed request                                |
| `401`  | Missing or invalid authentication                           |
| `404`  | Requested resource does not exist                           |
| `409`  | Conflict with the current resource state or idempotency key |
| `422`  | Request is valid JSON but contains invalid order data       |
| `429`  | Rate limit exceeded                                         |
| `500`  | Unexpected server error                                     |

## Pagination

Collection endpoints use cursor-based pagination.

Clients can control the number of results using `limit` and continue retrieving results using the returned `next_cursor`.

Example:

```http
GET /orders?limit=50&cursor=eyJvZmZzZXQiOjUwfQ==
```

Cursor-based pagination was chosen over page numbers because it is better suited to datasets such as orders that may change while a client is iterating through them.

## Rate Limiting

The API defines rate-limit information through response headers:

```text
X-RateLimit-Limit
X-RateLimit-Remaining
X-RateLimit-Reset
```

When the limit is exceeded, the API returns `429 Too Many Requests` and includes a `Retry-After` header indicating when the client should retry.

The OpenAPI specification defines the rate-limiting contract without prescribing a specific production quota. Actual limits would depend on infrastructure capacity and client requirements.

## Order Lifecycle

Orders may have the following statuses:

```text
open
partially_filled
filled
cancelled
rejected
```

Open and partially filled orders may be cancelled through:

```http
DELETE /orders/{order_id}
```

Filled or already cancelled orders cannot be cancelled and result in `409 Conflict`.

## Assumptions and Scope

This project focuses on the **API contract and system design**, rather than implementing a matching engine or brokerage backend.

The design assumes:

* Clients have already been authenticated and provisioned with an API key.
* Instrument data is supplied by an underlying market-data system.
* Order execution and matching are handled by an underlying trading system.
* Positions are calculated from executed trades.
* Monetary values are represented as numbers for simplicity in this API exercise. A production trading system would require a carefully defined decimal/fixed-point representation and precision policy.
* Authorization, account-level risk checks, compliance controls, persistence, and market connectivity would be responsibilities of the underlying implementation.

## Validation

The specification is written using **OpenAPI 3.1** and can be validated with Redocly CLI:

```bash
npx @redocly/cli lint openapi.yaml
```

## Repository Structure

```text
akuna-api-design/
├── openapi.yaml    # OpenAPI 3.1 API specification
└── README.md       # Design overview and documentation
```

## Possible Extensions

Given additional time, the API could be expanded with:

* Order modification
* Execution and trade history
* Account and buying-power information
* More advanced order types
* WebSocket-based market data and order updates
* Account-specific authorization and risk limits
* Request signing for stronger authentication
* Automated contract and integration testing

## Author

Thomas Shotts
