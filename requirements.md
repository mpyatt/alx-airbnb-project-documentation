# Requirement Specifications

This document details the technical and functional requirements for three core backend features of the Airbnb Clone project: **User Authentication**, **Property Management**, and **Booking System**.

---

## 1. User Authentication

### API Endpoints

| Method | Endpoint             | Description                      |
| ------ | -------------------- | -------------------------------- |
| POST   | `/api/auth/register` | Register a new user (guest/host) |
| POST   | `/api/auth/login`    | Authenticate and issue JWT       |
| POST   | `/api/auth/logout`   | Revoke refresh token             |
| GET    | `/api/auth/profile`  | Retrieve current user profile    |

### Input / Output

#### `POST /api/auth/register`

**Request Body** (JSON):

```json
{
  "email": "user@example.com",
  "password": "P@ssw0rd!",
  "role": "guest"         // or "host"
}
```

**Response (201 Created)**:

```json
{
  "id": "uuid-v4",
  "email": "user@example.com",
  "role": "guest",
  "access_token": "<JWT>",
  "refresh_token": "<token>"
}
```

#### `POST /api/auth/login`

**Request Body**:

```json
{
  "email": "user@example.com",
  "password": "P@ssw0rd!"
}
```

**Response (200 OK)**:

```json
{
  "access_token": "<JWT>",
  "refresh_token": "<token>",
  "expires_in": 900             // seconds
}
```

#### `GET /api/auth/profile`

**Headers**: `Authorization: Bearer <JWT>`

**Response (200 OK)**:

```json
{
  "id": "uuid-v4",
  "email": "user@example.com",
  "role": "guest",
  "created_at": "2025-06-29T12:00:00Z"
}
```

### Validation Rules

* **Email**: Required, valid format, unique.
* **Password**: Required, min length 8, at least one uppercase, one lowercase, one number, one special character.
* **Role**: Must be either `guest` or `host`.

### Security & Performance

* **Rate Limiting**: Max 5 login attempts per minute per IP.
* **Password Storage**: Bcrypt (cost factor 12).
* **Tokens**: Access token TTL = 15 minutes; Refresh token TTL = 7 days.
* **Response Time**: ≤200 ms under 100 RPS (95th percentile).

---

## 2. Property Management

### API Endpoints

| Method | Endpoint                       | Description                        |
| ------ | ------------------------------ | ---------------------------------- |
| GET    | `/api/properties`              | List all properties (with filters) |
| GET    | `/api/properties/{propertyId}` | Retrieve a single property         |
| POST   | `/api/properties`              | Create a new property listing      |
| PUT    | `/api/properties/{propertyId}` | Update an existing property        |
| DELETE | `/api/properties/{propertyId}` | Delete (soft) a property listing   |

### Input / Output

#### `GET /api/properties`

**Query Parameters**:

* `page` (integer, default=1)
* `per_page` (integer, default=20, max=100)
* `location` (string)
* `price_min`, `price_max` (number)
* `guests` (integer)
* `amenities` (comma-separated strings)

**Response (200 OK)**:

```json
{
  "data": [ { /* property object */ }, ... ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total_items": 125,
    "total_pages": 7
  }
}
```

#### `POST /api/properties`

**Request Body** (JSON):

```json
{
  "title": "Cozy Cottage",
  "description": "A lovely retreat...",
  "address": "123 Main St, City, Country",
  "latitude": 12.3456,
  "longitude": -65.4321,
  "price_per_night": 120.00,
  "currency": "USD",
  "amenities": ["wifi", "pool", "pet-friendly"],
  "availability": {
    "start_date": "2025-07-01",
    "end_date": "2025-12-31"
  },
  "images": ["https://.../img1.jpg"]
}
```

**Response (201 Created)**:

```json
{ /* newly created property object with id, timestamps, ownerId */ }
```

### Validation Rules

* **Title**: Required, max length 100.
* **Description**: Required.
* **Address/Geo**: Required.
* **Price**: ≥0.
* **Currency**: ISO 4217 code.
* **Availability Dates**: `start_date` < `end_date`.
* **Images**: URLs must be reachable and valid MIME types.

### Performance & Scalability

* **List Endpoint**: ≤300 ms under 200 RPS.
* **Create/Update**: ≤500 ms under 50 RPS.
* **Indexing**: Ensure geospatial and full-text indexes on `address`, `title`, and `amenities`.
* **Caching**: Optional Redis cache for popular search queries.

---

## 3. Booking System

### API Endpoints

| Method | Endpoint                                | Description                          |
| ------ | --------------------------------------- | ------------------------------------ |
| POST   | `/api/bookings`                         | Create a new booking                 |
| PATCH  | `/api/bookings/{bookingId}`             | Update booking status (e.g., cancel) |
| GET    | `/api/bookings/{bookingId}`             | Retrieve booking details             |
| GET    | `/api/users/{userId}/bookings`          | List all bookings for a given user   |
| GET    | `/api/properties/{propertyId}/bookings` | List all bookings for a property     |

### Input / Output

#### `POST /api/bookings`

**Request Body**:

```json
{
  "property_id": "uuid-v4",
  "user_id": "uuid-v4",
  "start_date": "2025-08-10",
  "end_date": "2025-08-15"
}
```

**Response (201 Created)**:

```json
{
  "id": "uuid-v4",
  "status": "pending",
  "total_cost": 600.00,
  "created_at": "2025-06-29T14:00:00Z"
}
```

#### `PATCH /api/bookings/{bookingId}`

**Request Body**:

```json
{ "status": "canceled" }
```

**Response (200 OK)**: Updated booking object.

### Validation Rules

* **Date Overlap**: New booking date range must not overlap any existing `confirmed` booking on the same property. Use database transaction + row-level lock.
* **User Role**: Only `guest` role may create bookings.
* **Cancellation**: Allowed only if `start_date` > current date and within policy window.

### Performance & Concurrency

* **Create Booking**: ≤400 ms under 100 concurrent requests.
* **Locking**: Employ optimistic locking or serializable transactions to avoid double-bookings.
* **Scaling**: Horizontal scaling behind a load balancer; sticky sessions not required for stateless API.

---

*Document last updated: 2025-06-29*
