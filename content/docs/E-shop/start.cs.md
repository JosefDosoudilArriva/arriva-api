---
title: Průvodce integrací s externím API
type: docs
prev: docs/folder/
---


## 1. Příprava

Před provedením jakýchkoli API volání je nutné získat bearer token. Tento token lze získat odesláním `POST` požadavku na `/v1/Authentication/ExtAuthenticate` s uživatelským jménem a heslem v těle požadavku.

- **Autentizace**: Získání bearer tokenu pomocí `POST` požadavku na `/v1/Authentication/ExtAuthenticate` s vaším uživatelským jménem a heslem v těle požadavku.

Pro práci s API je třeba získat několik konstant, včetně seznamů míst, tarifů a dalších. Konkrétně je třeba volat:

- `/v1/Constants/ExtAllPlaceTypes`
- `/v1/Constants/ExtAllPlaces`
- `/v1/Constants/ExtAllCompanies`
- `/v1/Constants/AllPassengerCategories`

## 2. Proces nákupu

### Krok 1: Autentizace

```plaintext
POST /v1/Authentication/ExtAuthenticate
```

- Odeslání uživatelského jména a hesla v požadavku.
- Obdržení bearer tokenu v odpovědi, který se použije pro všechna následující volání.

### Krok 2: Hledání míst

- Vyhledávání místa odjezdu a příjezdu, například "Praha" (id: 2236), "Příbram" (id: 3062).
- Ukládání ID míst pro vyhledávání spojení.
- Nastavení `limit: 10` pro omezení počtu výsledků.

### Krok 3: Hledání tras

```plaintext
POST /v1/Route/ExtSearch
```

- Požadované parametry: `sourcePlaceId`, `targetPlaceId` a `date`.
- Nepovinné parametry lze odstranit, pokud nejsou potřeba.
- Příklad zprávy:

```json
{
  "sourcePlaceId": 2236,
  "targetPlaceId": 3062,
  "dateTime": "2023-11-08T10:34:22.791Z"
}
```

- Obdržení `searchId` v odpovědi.

### Krok 4: Zahájení procesu nákupu

```plaintext
POST /v1/Purchase/ExtBegin
```
- Zahájení procesu nákupu a obdržení různých ID, včetně `purchaseId`.

### Krok 5: Nastavení cestujících ke spojení

```plaintext
POST /v1/Purchase/ExtSetPassengersToConnection
```

- Požadované parametry: `purchaseId`, `searchId` a `passengers`.
- Informace o cestujících jsou získány z `/v1/Constants/AllPassengerCategories`.

### Krok 6: Vytvoření rezervací

```plaintext
POST /v1/Reservation/ExtMakeReservations
```

- Vytvoření rezervací pomocí dat získaných z `RouteSearch`.

### Krok 7: Nastavení rezervací do košíku

```plaintext
POST /v1/Purchase/ExtSetReservationsToCart
```

- Přidání vytvořených rezervací do konkrétního košíku (`cartId`) pro konkrétní spojení (`searchId`).

### Krok 8: Označení nákupu jako placený

```plaintext
POST /v1/Purchase/PurchasePaid
```

- Označení objednávky jako zaplacené pomocí `purchaseId`.

### Krok 9: Získání objednávky

GET /v1/Purchase/GetOrder
```
- Získání podrobností objednávky pomocí `orderId`.