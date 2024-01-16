---
title: External API Integration Guide
type: docs
prev: docs/folder/
---


## 1. Preparation

Before making any API calls, you need to have a bearer token. This can be obtained by calling the authentication endpoint.

- **Authentication**: Obtain a bearer token by sending a `POST` request to `/v1/Authentication/ExtAuthenticate` with your username and password in the request body.

For working with the API, you need to retrieve several constants, including lists of places, tariffs, etc. Specifically, you need to call:

- `/v1/Constants/ExtAllPlaceTypes`
- `/v1/Constants/ExtAllPlaces`
- `/v1/Constants/ExtAllCompanies`
- `/v1/Constants/AllPassengerCategories`

## 2. Purchase Process

### Step 1: Authenticate

```plaintext
POST /v1/Authentication/ExtAuthenticate
```

- Send username and password in the request.
- Receive a bearer token in the response to use for all subsequent calls.

### Step 2: Search for Places

```plaintext
GET /v1/Place/ExtSearch
```

- Search for the place of departure and arrival, e.g., "Praha" (id: 2236), "Pribram" (id: 3062).
- Store the place IDs for searching connections.
- Set `limit: 10` to restrict the number of results.

### Step 3: Search for Routes

```plaintext
POST /v1/Route/ExtSearch
```

- Required parameters: `sourcePlaceId`, `targetPlaceId`, and `date`.
- Optional parameters can be removed if not needed.
- Example payload:

```json
{
  "sourcePlaceId": 2236,
  "targetPlaceId": 3062,
  "dateTime": "2023-11-08T10:34:22.791Z"
}
```

- Receive a `searchId` in the response.

### Step 4: Begin Purchase Process

```plaintext
POST /v1/Purchase/ExtBegin
```

- Initiate the purchase process and receive various IDs, including `purchaseId`.

### Step 5: Set Passengers to Connection

```plaintext
POST /v1/Purchase/ExtSetPassengersToConnection
```

- Required parameters: `purchaseId`, `searchId`, and `passengers`.
- Passengers info is retrieved from `/v1/Constants/AllPassengerCategories`.

### Step 6: Make Reservations

```plaintext
POST /v1/Reservation/ExtMakeReservations
```

- Create reservations using data obtained from `RouteSearch`.

### Step 7: Set Reservations to Cart

```plaintext
POST /v1/Purchase/ExtSetReservationsToCart
```

- Add created reservations to a specific cart (`cartId`) for a specific connection (`searchId`).

### Step 8: Mark Purchase as Paid

```plaintext
POST /v1/Purchase/PurchasePaid
```

- Mark the order as paid using `purchaseId`.

### Step 9: Retrieve Order

```plaintext
GET /v1/Purchase/GetOrder
```

- Retrieve the order details using `orderId`.
