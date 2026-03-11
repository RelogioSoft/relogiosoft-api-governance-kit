# Data Representations

**Category:** Design
**Tags:** data-types, dates, times, timezones, money, currency, geospatial, identifiers, phone, email, iso-8601, geojson

---

## Summary of Rules

### Dates, Times, and Timezones

- Dates **MUST** be in ISO 8601 format: `YYYY-MM-DD`
- Times **MUST** be in ISO 8601 format: `hh:mm:ss`
- DateTimes **MUST** be in ISO 8601 UTC format: `YYYY-MM-DDThh:mm:ssZ`
- Timezone offsets **SHOULD** be included as `Z` (UTC) or `±hh:mm`
- Named timezones **MUST** be in IANA format: e.g. `Europe/London`

### Durations

- Durations **SHOULD** use [ISO 8601 duration format](https://en.wikipedia.org/wiki/ISO_8601#Durations): e.g. `PT1H30M`
- If the duration is always an integer of a single unit, an integer suffixed with the unit **SHOULD** be used if it improves readability

### Money / Currency

- Money **MUST** be represented as a number type in the **minor unit** of the currency (e.g. cents, not euros)
- Currency codes **MUST** be in [ISO 4217](https://en.wikipedia.org/wiki/ISO_4217) format: `USD`, `EUR`, `GBP`
- A sender **MAY** include a currency code alongside the amount

### Geospatial Data

- Geospatial data **MUST** use [GeoJSON format](https://tools.ietf.org/html/rfc7946) as defined in RFC 7946

### Email Addresses

- Email addresses **MUST** use the property name `emailAddress` (singular) or `emailAddresses` (plural)

### Phone Numbers

- Phone numbers **MUST** be represented as a `string`
- Phone numbers **SHOULD** use [E.164 format](https://www.itu.int/rec/T-REC-E.164/)

### Identifiers

- Identifiers **MUST** be represented as `string` type, even when the underlying type is numeric

---

## Identifiers

Always use `string` type for identifiers, regardless of the underlying type:

```yaml
# OpenAPI schema
customerId:
  type: string
  description: "The unique identifier of the customer."
```

```json
{ "customerId": "12345" }
```

**Reason:** Using `string` allows the server to change the underlying identifier type (e.g. integer → UUID) in the future without a breaking change to the API contract.

**Security:** If a server fails to parse an identifier from a string to its underlying type, it **MUST** return `403 Forbidden`. Returning `400 Bad Request` with details about the parse failure reveals the underlying type to attackers and facilitates enumeration.

---

## Dates, Times, and Timezones

### Format Summary

| Data type | Format | Example |
|-----------|--------|---------|
| Date only | `YYYY-MM-DD` | `2024-07-23` |
| Time only | `hh:mm:ss` | `14:30:00` |
| Time with offset | `hh:mm:ss±hh:mm` or `hh:mm:ssZ` | `14:30:00+02:00`, `12:30:00Z` |
| DateTime (UTC) | `YYYY-MM-DDThh:mm:ssZ` | `2024-07-23T12:30:00Z` |
| DateTime with offset | `YYYY-MM-DDThh:mm:ss±hh:mm` | `2024-07-23T14:30:00+02:00` |
| Named timezone | IANA timezone name | `Europe/London`, `America/New_York` |

### Date Schema Example

```yaml
created:
  type: string
  format: date
  description: "The date the record was created."
```

```json
{ "created": "2024-07-23" }
```

### DateTime with Named Timezone

When both a timestamp and a named timezone are needed, use two separate fields:

```yaml
scheduledAt:
  type: string
  format: date-time
  description: "The scheduled date and time in UTC with offset."
timezone:
  type: string
  description: "The IANA timezone name for the scheduled time."
```

```json
{
  "scheduledAt": "2024-07-23T11:00:00+01:00",
  "timezone": "Europe/London"
}
```

### Availability / Unspecified DateTime

When storing an availability window in a local timezone (without UTC conversion):

```yaml
dateTime:
  type: string
  format: date-time
  description: "Local date-time without timezone conversion."
timeZone:
  type: string
  description: "IANA timezone name for interpreting the dateTime value."
```

```json
{
  "dateTime": "2024-07-23T12:30:00",
  "timezone": "Europe/Berlin"
}
```

---

## Durations

Use [ISO 8601 duration format](https://en.wikipedia.org/wiki/ISO_8601#Durations):

| Format | Meaning | Example |
|--------|---------|---------|
| `PT{n}S` | n seconds | `PT30S` |
| `PT{n}M` | n minutes | `PT5M` |
| `PT{n}H` | n hours | `PT2H` |
| `P{n}D` | n days | `P7D` |
| `P{n}M` | n months | `P3M` |
| `P{n}Y` | n years | `P1Y` |
| Combined | mixed units | `PT1H30M` |

**Exception:** If the duration is always an integer of a single well-known unit, a simpler representation is acceptable:

```json
{ "driveTimeMinutes": 5 }         // integer + unit in property name
{ "driveTime": "PT5M" }           // ISO 8601 preferred
{ "contractLength": "P12M" }      // 12 months
```

---

## Money / Currency

### Why Minor Units?

Store money as an integer in the minor currency unit (e.g. cents, pence) rather than as a decimal float.

**Reason:** Floating-point arithmetic cannot represent decimal fractions exactly in binary:

```
0.1 + 0.2 = 0.30000000000000004   // floating-point rounding error
```

Using integers avoids rounding errors entirely. `€51.22` is stored as `5122` cents.

### Schema Example

```yaml
total:
  type: number
  format: integer
  description: "The order total in the smallest unit of the relevant currency (e.g. cents for EUR/USD)."
currency:
  type: string
  description: "ISO 4217 currency code."
```

```json
{
  "total": 5122,
  "currency": "EUR"
}
```

### Multi-Currency

When multiple money values share the same currency, place the currency code at the parent level:

```json
{
  "currency": "USD",
  "subtotal": 4500,
  "tax": 405,
  "total": 4905
}
```

### Currency Code Optionality

Currency code **MAY** be omitted when:
- The platform operates in a single currency.
- The currency is deterministic from the request context (e.g. tenant/locale).

---

## Geospatial Data

Geospatial data **MUST** use [GeoJSON format](https://tools.ietf.org/html/rfc7946) (RFC 7946).

### Point

```yaml
type:
  type: string
  enum: [Point]
  description: "GeoJSON geometry type."
coordinates:
  type: array
  minItems: 2
  maxItems: 2
  items:
    type: number
    format: decimal
  description: "Longitude and latitude, in that order."
```

```json
{
  "type": "Point",
  "coordinates": [13.404954, 52.520007]
}
```

**Note:** Coordinates are `[longitude, latitude]` — this is the GeoJSON convention (not latitude/longitude).

### Polygon

```json
{
  "type": "Polygon",
  "coordinates": [
    [
      [100.0, 0.0], [101.0, 0.0], [101.0, 1.0],
      [100.0, 1.0], [100.0, 0.0]
    ]
  ]
}
```

For polygons with holes, the first ring is the exterior boundary; subsequent rings are interior holes.

### Query String Representation

In query strings, a point is a comma-separated `latitude,longitude` string:

```http
GET /locations/search?latlong=52.520007,13.404954
```

**Note:** In query strings, latitude comes first (opposite to GeoJSON body format).

```yaml
name: latlong
in: query
description: "Latitude and longitude of the search location (lat,long)."
schema:
  type: array
  minItems: 2
  maxItems: 2
  items:
    type: number
    format: decimal
example: "52.520007,13.404954"
```

---

## Phone Numbers

Phone numbers **MUST** be `string` type and **SHOULD** use [E.164 format](https://www.itu.int/rec/T-REC-E.164/):

```json
{ "phoneNumber": "+491234567890" }
```

E.164 format: `+` followed by country code and subscriber number, no spaces or separators. Maximum 15 digits.

---

## Email Addresses

Use `emailAddress` (not `email`) as the property name to avoid ambiguity with email message objects:

**Single address:**

```json
{ "emailAddress": "jane.doe@example.com" }
```

**Multiple addresses:**

```json
{
  "emailAddresses": [
    "jane.doe@example.com",
    "john.doe@example.com"
  ]
}
```

---

## OpenAPI Data Types Reference

For all other data types, follow the [OpenAPI 3.1 specification](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.1.0.md#data-types):

| OpenAPI Type | Format | Description |
|-------------|--------|-------------|
| `string` | — | General text |
| `string` | `date` | ISO 8601 date: `YYYY-MM-DD` |
| `string` | `date-time` | ISO 8601 date-time: `YYYY-MM-DDThh:mm:ssZ` |
| `string` | `time` | ISO 8601 time |
| `string` | `duration` | ISO 8601 duration |
| `string` | `uuid` | UUID format |
| `string` | `uri` | URI format |
| `string` | `email` | Email address (but use `emailAddress` as property name) |
| `number` | `integer` | Integer number (use for money in minor units) |
| `number` | `float` | Single-precision float |
| `number` | `double` | Double-precision float |
| `boolean` | — | `true` / `false` |
| `array` | — | Array of items |
| `object` | — | JSON object |
