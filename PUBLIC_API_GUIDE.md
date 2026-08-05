# Public API Guide

This document describes the partner API implemented by the `PublicApi` module, including its token model, authentication flow, scopes, management operations, and external endpoints. The core contracts and token lifecycle live under `src/api/PublicApi`; HTTP endpoints and middleware live under `src/api/Web/PublicApi`; persistence lives under `src/api/Infrastructure/PublicApi`.

## Purpose

The Public API lets an external partner work with products and orders for exactly one eCube business. Access is granted through a scoped API key. Each key:

- belongs to one business;
- has an explicit set of permissions;
- can have its own requests-per-minute limit and expiry;
- can be disabled, revoked, rotated, or soft-deleted;
- stores only a SHA-256 hash of the full key;
- can see only orders created through that same key.

## Project structure

```text
src/api/PublicApi/
├── Contracts/
│   ├── ApiScopes.cs                 Grantable scope catalog
│   ├── DTOs/                        Token create/update/response contracts
│   ├── Entities/ApiToken.cs         Persisted token model and usability rules
│   ├── Repositories/                Persistence abstraction
│   └── Services/                    Token service abstraction
└── Core/
    └── Services/
        ├── ApiKeyGenerator.cs       Secure key generation, parsing, and hashing
        └── ApiTokenService.cs       Token lifecycle and authentication logic

src/api/Web/PublicApi/               Admin and partner HTTP surface
src/api/Infrastructure/PublicApi/    EF Core configuration and repository
```

Both PublicApi projects target .NET 10. `Contracts` defines the stable boundary; `Core` implements it without depending on the web or EF Core projects.

## Authentication

External routes are under `/external` and require an API key using either header form:

```http
X-Api-Key: mrb_<prefix>_<secret>
```

```http
Authorization: Api-Key mrb_<prefix>_<secret>
```

Keys have this format:

```text
mrb_<10-character base62 prefix>_<43-character base62 secret>
```

The prefix provides an indexed database lookup. The service then hashes the complete presented key with SHA-256 and compares it to the stored hash using a constant-time comparison. The plaintext key is returned only when the token is created or rotated and cannot be retrieved later.

A key authenticates only when all of these conditions are true:

- its hash matches;
- `IsActive` is `true`;
- it has not been revoked;
- it has not expired;
- it has not been soft-deleted.

Successful authentication places the resolved `ApiToken` in `HttpContext.Items`. Controllers derive the business identity from this token; callers cannot use a request value to switch businesses.

`LastUsedAt` is updated on a best-effort basis at most once per minute. A failure to record usage does not fail the partner request.

## Scopes

Scopes are persisted as strings and validated against `ApiScopes.Catalog`.

| Scope | Allows |
|---|---|
| `products:list` | Search and list products |
| `products:find` | Get one product by ID |
| `orders:list` | Search and list orders created with this key |
| `orders:find` | Get one order created with this key |
| `orders:create` | Create an order |

Missing a required scope returns HTTP `403` with the standard external error shape.

## External endpoints

All list endpoints use a fixed page size of 25. A `page` value below 1 is treated as 1.

| Method | Route | Required scope | Description |
|---|---|---|---|
| `GET` | `/external/v1/products` | `products:list` | Search and filter products |
| `GET` | `/external/v1/products/{id}` | `products:find` | Find a product by GUID |
| `GET` | `/external/v1/orders` | `orders:list` | List orders created through the current key |
| `GET` | `/external/v1/orders/{orderNumberOrId}` | `orders:find` | Find an order by number or ID |
| `POST` | `/external/v1/orders` | `orders:create` | Create an order attributed to the current key |

### List products

`GET /external/v1/products` accepts these optional query parameters:

| Parameter | Type | Meaning |
|---|---|---|
| `query` | string | General product search |
| `page` | integer | One-based page number |
| `isAvailable` | boolean | Availability filter |
| `categoryIds` | comma-separated GUIDs | Category filter |
| `brandIds` | comma-separated GUIDs | Brand filter |
| `barcode` | string | Barcode filter |
| `sku` | string | SKU filter |

Malformed values inside `categoryIds` and `brandIds` are ignored. Product responses deliberately omit internal values such as buy price. `price` is the sale/list price, while `sellPrice` is the effective price after an active discount when present.

### List and find orders

`GET /external/v1/orders` accepts `query` and `page`. Both list and find operations enforce three boundaries: the order must belong to the token's business, have `ApiKey` as its source, and have been created using the current token ID. A partner key therefore cannot read storefront orders or orders created by another key for the same business.

### Create an order

Example request:

```json
{
  "customerName": "Example Customer",
  "customerPhone": "+9647700000000",
  "note": "Leave at reception",
  "items": [
    {
      "productId": "11111111-1111-1111-1111-111111111111",
      "combinationId": null,
      "quantity": 2
    }
  ],
  "address": {
    "lineOne": "Street 1",
    "city": "Baghdad",
    "country": "Iraq",
    "phone": "+9647700000000"
  }
}
```

`customerName`, `customerPhone`, and at least one item are required. Every item needs a non-empty `productId` and a quantity greater than zero. The address phone falls back to `customerPhone`. The operation is rejected while the store is under maintenance.

Created orders are marked with source `ApiKey` and the authenticated token's ID. Customer lookup reuses an existing customer ID when the phone matches; otherwise it generates an ID while the order retains the submitted customer details.

## Pagination response

List endpoints return:

```json
{
  "items": [],
  "page": 1,
  "pageSize": 25,
  "total": 0
}
```

## Errors and rate limiting

Every external API failure uses the same body:

```json
{
  "error": "machine_readable_code",
  "message": "Human-readable explanation"
}
```

Common statuses are:

| Status | Meaning |
|---|---|
| `400` | Invalid request or order creation failure |
| `401` | Missing, malformed, inactive, revoked, expired, or incorrect API key |
| `403` | Token lacks the required scope |
| `404` | Product or permitted order not found |
| `429` | Per-token request limit exceeded |
| `500` | Unexpected server error |
| `503` | Store is under maintenance |

Rate limiting uses a fixed one-minute window partitioned by token ID, with no queue. `ApiToken.RateLimitPerMinute` overrides the global `PublicApi:DefaultRateLimitPerMinute` setting, whose fallback is 60. A `429` response includes a `Retry-After` header.

## Admin token endpoints

Token management is restricted to Morabaa staff and is separate from API-key authentication.

| Method | Route | Description |
|---|---|---|
| `POST` | `/admin/api-tokens` | Create a token and reveal its key once |
| `GET` | `/admin/api-tokens/scopes` | List grantable scopes and descriptions |
| `GET` | `/admin/api-tokens?businessId={id}` | List tokens, optionally by business |
| `GET` | `/admin/api-tokens/{id}` | Get token metadata |
| `PATCH` | `/admin/api-tokens/{id}` | Partially update a token |
| `POST` | `/admin/api-tokens/{id}/rotate` | Replace the secret and reveal the new key once |
| `POST` | `/admin/api-tokens/{id}/revoke` | Permanently mark a token revoked and inactive |
| `DELETE` | `/admin/api-tokens/{id}` | Soft-delete a token |

Create validation requires a non-empty business ID, a non-blank name, at least one valid scope, and a rate limit of at least 1 when supplied. Scope values are deduplicated before persistence.

The update contract is partial: only non-null values are applied. In the current implementation, this means `RateLimitPerMinute` and `ExpiresAt` can be assigned but cannot be cleared back to `null` through the PATCH endpoint. A blank `Name` is ignored rather than applied.

Rotation immediately invalidates the old plaintext key by replacing its prefix and hash. Revocation sets both `RevokedAt` and `IsActive = false`. Deletion uses the inherited soft-delete behavior, preserving the database row.

Normal token metadata responses never contain `Key` or `KeyHash`. `CreatedApiTokenResponse` adds the plaintext `key` only to create and rotate responses.

## Persistence

`ApiToken` is stored through the shared CRUD repository infrastructure. EF Core configures:

- a unique index on `KeyPrefix` for authentication lookup;
- indexes on `BusinessId` and `DeletedAt`;
- maximum lengths of 200 for `Name`, 64 for `KeyPrefix`, and 128 for `KeyHash`;
- scopes in one comma-separated database column with a list value comparer.

Because scope storage is comma-separated, future scope names must not contain commas.

## Request pipeline and registration

`AddPublicApi()` registers the repository, token service, and named rate-limit policy. Middleware order is significant:

1. `UsePublicApiExceptions()` standardizes failures for `/external` routes.
2. `UsePublicApiAuth()` validates the key and attaches its token.
3. ASP.NET rate limiting reads that token to partition and configure the limit.
4. MVC scope filters authorize the selected controller action.

Requests outside `/external` pass through the Public API authentication and exception middleware unchanged.
