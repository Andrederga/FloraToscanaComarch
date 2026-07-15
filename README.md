# SAP — Postman Collection

Postman collection for interacting with the **SAP S/4HANA Cloud** OData APIs. It covers reading Business Partners and the full sales process (Sales Flow): order → delivery → picking → goods issue → billing.

---

## Table of contents

- [Requirements](#requirements)
- [Authentication](#authentication)
- [Variables](#variables)
- [CSRF token handling](#csrf-token-handling)
- [Endpoints](#endpoints)
  - [Business Partner](#business-partner)
  - [Sales Flow](#sales-flow)
- [End-to-end sales flow](#end-to-end-sales-flow)
- [Notes](#notes)

---

## Requirements

- **Postman** (v10 or later recommended).
- A reachable **SAP S/4HANA Cloud** tenant and its credentials (communication user).
- The communication scenarios enabled on the tenant for the APIs used:
  - `API_BUSINESS_PARTNER`
  - `API_SALES_ORDER_SRV` (OData V4)
  - `API_OUTBOUND_DELIVERY_SRV;v=2`
  - `API_BILLING_DOCUMENT_SRV` (OData V4)

---

## Authentication

Authentication is configured **at the collection level** as **Basic Auth** and is inherited by every request:

```
type: basic
username: {{username}}
password: {{password}}
```

Individual requests must not redefine the `Authorization` header: they inherit it automatically from the collection.

---

## Variables

Set these variables before running the requests (at the **environment** or **collection** level).

| Variable     | Scope       | Description                                                              | Example                                             |
|--------------|-------------|-------------------------------------------------------------------------|-----------------------------------------------------|
| `DEV_URL`    | environment | Base URL of the SAP tenant (no trailing slash).                         | `https://my427213-api.s4hana.cloud.sap`             |
| `username`   | environment | SAP communication user.                                                 | `COMM_USER`                                         |
| `password`   | environment | Password of the communication user.                                     | `••••••••`                                          |
| `csrfToken`  | collection  | CSRF token. **Populated automatically** by the pre-request script.      | *(auto)*                                           |

> `csrfToken` already exists in the collection with an empty value: do not fill it in manually.

---

## CSRF token handling

Every **POST** request in the Sales Flow includes a **pre-request** script that automatically fetches the CSRF token before sending. Script logic:

1. Runs only when `pm.request.method === "POST"`.
2. Resolves the variables in the request URL to get the real URL.
3. Derives the **service root** by cutting the URL after the `/0001/` segment; as a fallback it strips the last entity from the path.
4. Builds an `Authorization: Basic ...` header from `username` and `password`.
5. Sends a `GET` to the service root with the header `X-CSRF-Token: Fetch`.
6. If the response returns a valid token, it stores it in `csrfToken` and applies it to the current request's `X-CSRF-Token` header.

This means you never need to fetch the token manually: it is enough that the credentials and `DEV_URL` are correct.

---

## Endpoints

### Business Partner

#### Read
Reads the list of Business Partners (first 50 records).

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `{{DEV_URL}}/sap/opu/odata/sap/API_BUSINESS_PARTNER/A_BusinessPartner?$top=50` |
| **Headers** | `Content-Type: application/json`, `Accept: application/json` |
| **Query** | `$top=50` — limits the number of returned records |

---

### Sales Flow

All requests below are **POST** with `Content-Type: application/json` and `Accept: application/json` headers, and use the pre-request script for the CSRF token.

#### 1. Create Sales Order
Creates a sales order (OData **V4**).

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `{{DEV_URL}}/sap/opu/odata4/sap/api_salesorder/srvd_a2x/sap/salesorder/0001/SalesOrder` |

**Body (example):**

```json
{
  "SalesOrderType": "TA",
  "SoldToParty": "1000006",
  "SalesOrganization": "PF10",
  "DistributionChannel": "GD",
  "OrganizationDivision": "10",
  "PurchaseOrderByCustomer": "Created via ODV4 API",
  "_Item": [
    {
      "Product": "DISTDIRS065SA1",
      "RequestedQuantity": 10,
      "RequestedQuantityISOUnit": "PCE",
      "_ItemPricingElement": [
        {
          "ConditionType": "PMP0",
          "ConditionRateAmount": 45.98,
          "ConditionCurrency": "EUR"
        }
      ]
    }
  ]
}
```

**Key fields:**

| Field | Description |
|---|---|
| `SalesOrderType` | Order type (`TA` = standard order) |
| `SoldToParty` | Sold-to customer |
| `SalesOrganization` / `DistributionChannel` / `OrganizationDivision` | Sales area |
| `_Item` | Order line items (product, quantity, ISO unit) |
| `_ItemPricingElement` | Item pricing conditions (type, amount, currency) |

> The ID of the created order is used as the reference (`ReferenceSDDocument`) for the following delivery.

#### 2. Create Outbound Delivery
Creates the outbound delivery from the order (OData **V2**).

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `{{DEV_URL}}/sap/opu/odata/sap/API_OUTBOUND_DELIVERY_SRV;v=2/A_OutbDeliveryHeader` |

**Body (example):**

```json
{
  "ShippingPoint": "PF10",
  "to_DeliveryDocumentItem": {
    "results": [
      { "ReferenceSDDocument": "287" }
    ]
  }
}
```

> Replace `ReferenceSDDocument` with the sales order number created in step 1.

#### 3. Delivery Picking
Picks all delivery items (action `PickAllItems`).

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `{{DEV_URL}}/sap/opu/odata/sap/API_OUTBOUND_DELIVERY_SRV;v=2/PickAllItems?DeliveryDocument='80000001'` |
| **Query** | `DeliveryDocument='80000001'` — delivery number (to be replaced) |
| **Body** | empty — the action uses only the query parameter |

> Update `DeliveryDocument` with the actual delivery obtained in step 2.

#### 4. Delivery Goods Issue
Posts the goods issue of the delivery (action `PostGoodsIssue`).

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `{{DEV_URL}}/sap/opu/odata/sap/API_OUTBOUND_DELIVERY_SRV;v=2/PostGoodsIssue?DeliveryDocument='80000001'` |
| **Query** | `DeliveryDocument='80000001'` — delivery number (to be replaced) |
| **Body** | empty — the action uses only the query parameter |

> Final logistics step: after picking, post the goods issue from the warehouse. Update `DeliveryDocument` with the actual delivery number.

#### 5. Create Billing Document from reference document
Creates the billing document from the reference SD document (OData **V4**, action `CreateFromSDDocument`).

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `{{DEV_URL}}/sap/opu/odata4/sap/api_billingdocument/srvd_a2x/sap/billingdocument/0001/BillingDocument/SAP__self.CreateFromSDDocument` |

**Body (example):**

```json
{
  "_Reference": [
    { "SDDocument": "80017574" }
  ]
}
```

> `SDDocument` can be a delivery or a sales order, depending on the billing configuration.

#### 6. Fetch token
Placeholder request to fetch the CSRF token manually.

| | |
|---|---|
| **Method** | `GET` |
| **URL** | *(empty — to be filled in)* |

> ⚠️ The URL is empty. To use it, set the service root and the header `X-CSRF-Token: Fetch`.

---

## End-to-end sales flow

Typical execution sequence for the Sales Flow requests:

1. **Create Sales Order** → note the returned order number.
2. **Create Outbound Delivery** → use the order number as `ReferenceSDDocument`; note the delivery number.
3. **Delivery Picking** → pass the delivery number in `DeliveryDocument`.
4. **Delivery Goods Issue** → post the goods issue for the delivery.
5. **Create Billing Document** → use the document (delivery/order) as `SDDocument`.

Each step depends on the identifier generated by the previous one: update the references in the body/query before running.

---

## Notes

Items to keep in mind for the collection:

- **Hardcoded values**: document numbers (`'80000001'`, `287`, `80017574`) and master data (customer, product, sales area) are sample values: replace them with the real ones from your tenant.
- **Fetch token**: the URL is empty; fill it in only if you want to retrieve the token manually instead of via the pre-request script.