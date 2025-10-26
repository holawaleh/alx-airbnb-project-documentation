# Backend Requirement Specifications – Airbnb Clone

This document outlines detailed functional and technical requirements for three core backend features of the Airbnb Clone: **User Authentication**, **Property Management**, and **Booking System**.

All APIs follow RESTful conventions ang Graphql, use JSON for data exchange, and return appropriate HTTP status codes.

---

## 1. User Authentication

### Functional Description
Allow users to register and log in as **Guests** or **Hosts**. Support secure session management via JWT and protect sensitive data.

### API Endpoints

#### `POST /api/auth/register`
- **Description**: Register a new user
- **Request Body**:
  ```json
  {
    "email": "user@example.com",
    "password": "SecurePass123!",
    "role": "guest" | "host",
    "full_name": "Amina Yusuf",
    "phone": "+2348012345678"
  }
  ```
- **Validation Rules**:
  - `email`: valid format, unique
  - `password`: min 8 chars, at least 1 uppercase, 1 number
  - `role`: required, must be "guest" or "host"
  - `phone`: Nigerian format (e.g., +234...)
- **Success Response (201)**:
  ```json
  {
    "id": "usr_abc123",
    "email": "user@example.com",
    "role": "guest",
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
  ```
- **Error Responses**:
  - `400`: Invalid input
  - `409`: Email already exists

#### `POST /api/auth/login`
- **Description**: Authenticate user and issue JWT
- **Request Body**:
  ```json
  {
    "email": "user@example.com",
    "password": "SecurePass123!"
  }
  ```
- **Success Response (200)**:
  ```json
  {
    "id": "usr_abc123",
    "email": "user@example.com",
    "role": "host",
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
  ```
- **Error Responses**:
  - `401`: Invalid credentials

### Security & Performance
- Passwords hashed with `bcrypt` (cost factor 12)
- JWT expires in **24 hours**
- Rate limiting: max 5 login attempts/minute per IP
- All endpoints protected against CSRF and XSS

---

## 2. Property Management

### Functional Description
Enable hosts to create, update, and delete property listings with rich metadata relevant to Nigerian rentals.

### API Endpoints

#### `POST /api/properties`
- **Description**: Create a new property listing (host only)
- **Authentication**: JWT required, role = `host`
- **Request Body**:
  ```json
  {
    "title": "Cozy Apartment in Lekki",
    "description": "Peaceful 2-bedroom with generator and borehole water",
    "location": {
      "city": "Lagos",
      "state": "Lagos",
      "country": "Nigeria",
      "lat": 6.4550,
      "lng": 3.3841
    },
    "price_per_night": 45000,
    "currency": "NGN",
    "max_guests": 4,
    "amenities": ["Wi-Fi", "Generator", "Parking", "Borehole Water"],
    "available_from": "2025-11-01",
    "available_to": "2026-12-31"
  }
  ```
- **Validation Rules**:
  - `price_per_night` ≥ 1000 NGN
  - `max_guests` ≥ 1
  - `amenities`: from predefined list
  - Coordinates required for map integration
- **Success Response (201)**:
  ```json
  {
    "id": "prop_xyz789",
    "host_id": "usr_abc123",
    "title": "Cozy Apartment in Lekki",
    "price_per_night": 45000,
    "currency": "NGN",
    "created_at": "2025-10-26T10:00:00Z"
  }
  ```

#### `PUT /api/properties/{id}`
- **Description**: Update own property (host only)
- **Authorization**: Host must own the property
- **Response (200)**: Updated property object

#### `DELETE /api/properties/{id}`
- **Description**: Delete own property
- **Response (204)**: No content

### Performance & Storage
- Property images stored in **Cloudinary** or **AWS S3**
- Max 10 images per listing
- Database indexed on `city`, `price_per_night`, `amenities` for fast search

---

## 3. Booking System

### Functional Description
Allow guests to book available properties for specific dates, prevent double bookings, and manage booking lifecycle.

### API Endpoints

#### `POST /api/bookings`
- **Description**: Create a new booking
- **Authentication**: JWT required, role = `guest`
- **Request Body**:
  ```json
  {
    "property_id": "prop_xyz789",
    "start_date": "2025-12-20",
    "end_date": "2025-12-25"
  }
  ```
- **Validation Rules**:
  - `start_date` < `end_date`
  - Dates in future
  - Property must exist and be active
  - Property must be **available** for all dates in range
  - Guest cannot book own property
- **Success Flow**:
  1. Check availability in `bookings` table
  2. Create booking with status = `pending`
  3. Trigger payment (via Paystack/Flutterwave)
  4. On payment success → status = `confirmed`
- **Success Response (201)**:
  ```json
  {
    "id": "book_456def",
    "property_id": "prop_xyz789",
    "guest_id": "usr_guest123",
    "start_date": "2025-12-20",
    "end_date": "2025-12-25",
    "status": "confirmed",
    "total_price": 225000
  }
  ```
- **Error Responses**:
  - `400`: Invalid dates
  - `409`: Dates not available (double-booking prevented)
  - `403`: Cannot book own property

#### `PATCH /api/bookings/{id}/cancel`
- **Description**: Cancel booking (guest or host)
- **Rules**:
  - Free cancellation if >48h before check-in
  - Refund processed via payment gateway
- **Response (200)**: Updated booking with `status: "canceled"`

### Data Integrity
- **Database constraint**: Unique `(property_id, date)` to prevent overlaps
- Use **transactional queries** during booking creation

### Performance
- Availability check optimized with **date-range indexing**
- Caching: Redis cache for popular property availability (TTL: 5 min)

---

> **Note**: All monetary values default to **NGN (₦)**. Multi-currency support is out of scope for MVP.