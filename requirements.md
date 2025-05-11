# Backend Requirements Specification

## 1. User Authentication System

### Overview
The User Authentication system will manage user registration, login, profile management, and session handling for guests, hosts, and administrators.

### Database Schema

#### Users Table
```sql
CREATE TABLE users (
    user_id CHAR(36) PRIMARY KEY,
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    phone_number VARCHAR(50),
    role ENUM('guest', 'host', 'admin') NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE INDEX idx_user_email (email)
);
```

### API Endpoints

#### User Registration
- **Endpoint**: `POST /api/auth/register`
- **Description**: Register a new user as either a guest or host
- **Request Body**:
  ```json
  {
    "email": "string",
    "password": "string",
    "firstName": "string",
    "lastName": "string",
    "role": "guest|host",
    "phoneNumber": "string (optional)",
    "profileImage": "file (optional)"
  }
  ```
- **Response**:
  - Success (201 Created):
    ```json
    {
      "userId": "integer",
      "email": "string",
      "firstName": "string",
      "lastName": "string",
      "role": "string",
      "message": "User registered successfully. Please verify your email."
    }
    ```
  - Error (400 Bad Request):
    ```json
    {
      "error": "string",
      "details": "object (validation errors)"
    }
    ```

#### User Login
- **Endpoint**: `POST /api/auth/login`
- **Description**: Authenticate user and issue JWT token
- **Request Body**:
  ```json
  {
    "email": "string",
    "password": "string"
  }
  ```
- **Response**:
  - Success (200 OK):
    ```json
    {
      "token": "string (JWT)",
      "userId": "integer",
      "email": "string",
      "firstName": "string",
      "role": "string",
      "expiresIn": "integer (seconds)"
    }
    ```
  - Error (401 Unauthorized):
    ```json
    {
      "error": "Invalid credentials"
    }
    ```

#### Email Verification
- **Endpoint**: `GET /api/auth/verify-email/:token`
- **Description**: Verify user email with verification token
- **Response**:
  - Success (200 OK):
    ```json
    {
      "message": "Email verified successfully"
    }
    ```
  - Error (400 Bad Request):
    ```json
    {
      "error": "Invalid or expired token"
    }
    ```

#### Password Reset Request
- **Endpoint**: `POST /api/auth/forgot-password`
- **Description**: Request password reset email
- **Request Body**:
  ```json
  {
    "email": "string"
  }
  ```
- **Response**:
  - Success (200 OK):
    ```json
    {
      "message": "If the email exists, a password reset link has been sent"
    }
    ```

#### Password Reset
- **Endpoint**: `POST /api/auth/reset-password/:token`
- **Description**: Reset password using token
- **Request Body**:
  ```json
  {
    "newPassword": "string"
  }
  ```
- **Response**:
  - Success (200 OK):
    ```json
    {
      "message": "Password reset successfully"
    }
    ```
  - Error (400 Bad Request):
    ```json
    {
      "error": "Invalid or expired token"
    }
    ```

#### Get User Profile
- **Endpoint**: `GET /api/users/profile`
- **Description**: Get current user's profile
- **Authentication**: Required
- **Response**:
  - Success (200 OK):
    ```json
    {
      "userId": "integer",
      "email": "string",
      "firstName": "string",
      "lastName": "string",
      "role": "string",
      "phoneNumber": "string",
      "profileImage": "string (URL)",
      "createdAt": "date"
    }
    ```
  - Error (401 Unauthorized):
    ```json
    {
      "error": "Unauthorized"
    }
    ```

#### Update User Profile
- **Endpoint**: `PUT /api/users/profile`
- **Description**: Update current user's profile
- **Authentication**: Required
- **Request Body**:
  ```json
  {
    "firstName": "string (optional)",
    "lastName": "string (optional)",
    "phoneNumber": "string (optional)",
    "profileImage": "file (optional)"
  }
  ```
- **Response**:
  - Success (200 OK):
    ```json
    {
      "message": "Profile updated successfully",
      "user": {
        "userId": "integer",
        "firstName": "string",
        "lastName": "string",
        "phoneNumber": "string",
        "profileImage": "string (URL)"
      }
    }
    ```

### Validation Rules

#### User Registration
- Email:
  - Must be a valid email format
  - Maximum length: 255 characters
  - Must be unique in the system
- Password:
  - Minimum length: 8 characters
  - Must include at least one uppercase letter, one lowercase letter, one number, and one special character
- First Name & Last Name:
  - Required
  - Maximum length: 100 characters each
  - Alphanumeric characters, spaces, and hyphens allowed
- Role:
  - Must be either "guest" or "host"
- Phone Number:
  - Optional for guests, required for hosts
  - Must be a valid phone number format
- Profile Image:
  - Optional
  - Maximum file size: 5MB
  - Allowed formats: JPEG, PNG

### Security Requirements
1. Passwords must be hashed using bcrypt with a work factor of at least 10
2. JWT tokens must expire after 24 hours
3. Implement rate limiting: 10 login attempts per IP address per hour
4. Session information should not be stored in localStorage (use HTTP-only cookies)
5. Implement CSRF protection for all API endpoints

### Performance Criteria
1. Authentication response time should be under 500ms (95th percentile)
2. Password hashing should use appropriate work factor to balance security and performance
3. System should handle up to 100 authentication requests per second

## 2. Property Management System

### Overview
The Property Management system will allow hosts to create, edit, and manage their property listings, including uploading photos, setting availability, and defining pricing.

### Database Schema

#### Properties Table
```sql
CREATE TABLE property (
    property_id CHAR(36) PRIMARY KEY,
    host_id CHAR(36) NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    location VARCHAR(255) NOT NULL,
    price_per_night DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_property_host (host_id),
    CONSTRAINT fk_property_host FOREIGN KEY (host_id) REFERENCES `users` (user_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);

CREATE TABLE PropertyAmenities (
  property_amenity_id SERIAL PRIMARY KEY,
  property_id INTEGER NOT NULL REFERENCES Properties(property_id),
  amenity_name VARCHAR(100) NOT NULL,
  amenity_category VARCHAR(50)
);

CREATE TABLE PropertyImages (
  image_id SERIAL PRIMARY KEY,
  property_id INTEGER NOT NULL REFERENCES Properties(property_id),
  image_url VARCHAR(255) NOT NULL,
  caption VARCHAR(255),
  is_primary BOOLEAN DEFAULT FALSE,
  upload_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE PropertyAvailability (
  availability_id SERIAL PRIMARY KEY,
  property_id INTEGER NOT NULL REFERENCES Properties(property_id),
  date DATE NOT NULL,
  is_available BOOLEAN DEFAULT TRUE,
  price_adjustment DECIMAL(10, 2) DEFAULT 0,
  minimum_stay INTEGER DEFAULT 1,
  UNIQUE (property_id, date)
);
```

### API Endpoints

#### Create Property Listing
- **Endpoint**: `POST /api/properties`
- **Description**: Create a new property listing
- **Authentication**: Required (Host only)
- **Request Body**:
  ```json
  {
    "title": "string",
    "description": "string",
    "propertyType": "string",
    "roomType": "string",
    "address": "string",
    "city": "string",
    "state": "string",
    "country": "string",
    "zipCode": "string",
    "pricePerNight": "number",
    "cleaningFee": "number",
    "serviceFee": "number",
    "maxGuests": "integer",
    "bedrooms": "integer",
    "beds": "integer",
    "bathrooms": "number",
    "amenities": ["string"],
    "images": ["file"]
  }
  ```
- **Response**:
  - Success (201 Created):
    ```json
    {
      "propertyId": "integer",
      "title": "string",
      "description": "string",
      "address": "string",
      "pricePerNight": "number",
      "message": "Property created successfully"
    }
    ```
  - Error (400 Bad Request):
    ```json
    {
      "error": "string",
      "details": "object (validation errors)"
    }
    ```

#### Update Property Listing
- **Endpoint**: `PUT /api/properties/{propertyId}`
- **Description**: Update an existing property listing
- **Authentication**: Required (Property owner only)
- **Request Body**: Same as Create Property (all fields optional)
- **Response**:
  - Success (200 OK):
    ```json
    {
      "propertyId": "integer",
      "title": "string",
      "message": "Property updated successfully"
    }
    ```
  - Error (403 Forbidden):
    ```json
    {
      "error": "You don't have permission to update this property"
    }
    ```

#### Get Property Details
- **Endpoint**: `GET /api/properties/{propertyId}`
- **Description**: Get detailed information about a property
- **Response**:
  - Success (200 OK):
    ```json
    {
      "propertyId": "integer",
      "hostId": "integer",
      "hostName": "string",
      "hostImage": "string (URL)",
      "title": "string",
      "description": "string",
      "propertyType": "string",
      "roomType": "string",
      "address": "string",
      "city": "string",
      "state": "string",
      "country": "string",
      "zipCode": "string",
      "latitude": "number",
      "longitude": "number",
      "pricePerNight": "number",
      "cleaningFee": "number",
      "serviceFee": "number",
      "maxGuests": "integer",
      "bedrooms": "integer",
      "beds": "integer",
      "bathrooms": "number",
      "amenities": [
        {
          "name": "string",
          "category": "string"
        }
      ],
      "images": [
        {
          "imageUrl": "string",
          "caption": "string",
          "isPrimary": "boolean"
        }
      ],
      "rating": "number",
      "reviewCount": "integer",
      "createdAt": "date"
    }
    ```

#### Delete Property Listing
- **Endpoint**: `DELETE /api/properties/{propertyId}`
- **Description**: Delete a property listing
- **Authentication**: Required (Property owner or Admin only)
- **Response**:
  - Success (200 OK):
    ```json
    {
      "message": "Property deleted successfully"
    }
    ```

#### Upload Property Images
- **Endpoint**: `POST /api/properties/{propertyId}/images`
- **Description**: Upload images for a property
- **Authentication**: Required (Property owner only)
- **Request Body**: Form data with multiple image files
- **Response**:
  - Success (200 OK):
    ```json
    {
      "message": "Images uploaded successfully",
      "images": [
        {
          "imageId": "integer",
          "imageUrl": "string",
          "isPrimary": "boolean"
        }
      ]
    }
    ```

#### Set Property Availability
- **Endpoint**: `POST /api/properties/{propertyId}/availability`
- **Description**: Set availability for a property
- **Authentication**: Required (Property owner only)
- **Request Body**:
  ```json
  {
    "dates": [
      {
        "date": "YYYY-MM-DD",
        "isAvailable": "boolean",
        "priceAdjustment": "number (optional)",
        "minimumStay": "integer (optional)"
      }
    ]
  }
  ```
- **Response**:
  - Success (200 OK):
    ```json
    {
      "message": "Availability updated successfully"
    }
    ```

#### Get Host Properties
- **Endpoint**: `GET /api/users/properties`
- **Description**: Get all properties for the authenticated host
- **Authentication**: Required (Host only)
- **Response**:
  - Success (200 OK):
    ```json
    {
      "properties": [
        {
          "propertyId": "integer",
          "title": "string",
          "status": "string",
          "pricePerNight": "number",
          "city": "string",
          "country": "string",
          "primaryImage": "string (URL)",
          "createdAt": "date"
        }
      ]
    }
    ```

### Validation Rules

#### Property Creation/Update
- Title:
  - Required
  - Length: 10-255 characters
- Description:
  - Required
  - Minimum length: 50 characters
- Property Type:
  - Required
  - Must be one of the predefined types (e.g., apartment, house, villa)
- Room Type:
  - Required
  - Must be one of the predefined types (e.g., entire place, private room, shared room)
- Address, City, State, Country:
  - All required
  - Maximum length: 100 characters each (except address: 255)
- Price per Night:
  - Required
  - Minimum: $1.00
  - Maximum: $10,000.00
- Cleaning & Service Fees:
  - Optional
  - Minimum: $0.00
  - Maximum: $1,000.00
- Guests, Bedrooms, Beds:
  - Required
  - Minimum: 1
  - Maximum: 50
- Bathrooms:
  - Required
  - Minimum: 0.5
  - Maximum: 20
- Images:
  - At least one image required
  - Maximum 20 images per property
  - Maximum file size: 10MB each
  - Allowed formats: JPEG, PNG, WebP

### Security Requirements
1. Only property owners and admins can modify or delete property listings
2. Secure file uploads with validation for file types and sizes
3. Implement geocoding validation to verify address authenticity
4. Sanitize all text inputs to prevent XSS attacks

### Performance Criteria
1. Property listing creation should complete within 3 seconds (including image uploads)
2. Property search queries should return results in under 500ms (95th percentile)
3. System should handle up to 50 property management operations per second
4. Images should be optimized and served via CDN

## 3. Booking System

### Overview
The Booking System will manage the entire process of creating, updating, and managing bookings between guests and hosts, including availability checking, payment processing, and booking status tracking.

### Database Schema

#### Bookings Table
```sql
CREATE TABLE `booking` (
    booking_id CHAR(36) PRIMARY KEY,
    property_id CHAR(36) NOT NULL,
    user_id CHAR(36) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    total_price DECIMAL(10, 2) NOT NULL,
    status ENUM('pending', 'confirmed', 'canceled') NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_booking_property (property_id),
    INDEX idx_booking_user (user_id),
    INDEX idx_booking_dates (start_date, end_date),
    CONSTRAINT fk_booking_property FOREIGN KEY (property_id) REFERENCES `property` (property_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE,
    CONSTRAINT fk_booking_user FOREIGN KEY (user_id) REFERENCES `users` (user_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);

CREATE TABLE BookingPayments (
  payment_id SERIAL PRIMARY KEY,
  booking_id INTEGER NOT NULL REFERENCES Bookings(booking_id),
  payment_intent_id VARCHAR(255),
  amount DECIMAL(10, 2) NOT NULL,
  currency VARCHAR(3) DEFAULT 'USD',
  payment_status VARCHAR(20) NOT NULL CHECK (payment_status IN ('pending', 'completed', 'failed', 'refunded', 'partially_refunded')),
  payment_method VARCHAR(50),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### API Endpoints

#### Check Property Availability
- **Endpoint**: `GET /api/properties/{propertyId}/availability`
- **Description**: Check if a property is available for specific dates
- **Query Parameters**:
  - `checkIn`: Date (YYYY-MM-DD)
  - `checkOut`: Date (YYYY-MM-DD)
- **Response**:
  - Success (200 OK):
    ```json
    {
      "available": "boolean",
      "unavailableDates": ["YYYY-MM-DD"],
      "basePrice": "number",
      "totalNights": "integer",
      "priceDetails": {
        "baseTotal": "number",
        "cleaningFee": "number",
        "serviceFee": "number",
        "total": "number"
      },
      "minimumStay": "integer"
    }
    ```

#### Create Booking
- **Endpoint**: `POST /api/bookings`
- **Description**: Create a new booking request
- **Authentication**: Required (Guest only)
- **Request Body**:
  ```json
  {
    "propertyId": "integer",
    "checkInDate": "YYYY-MM-DD",
    "checkOutDate": "YYYY-MM-DD",
    "numberOfGuests": "integer",
    "specialRequests": "string (optional)"
  }
  ```
- **Response**:
  - Success (201 Created):
    ```json
    {
      "bookingId": "integer",
      "status": "pending",
      "checkInDate": "YYYY-MM-DD",
      "checkOutDate": "YYYY-MM-DD",
      "priceDetails": {
        "baseTotal": "number",
        "cleaningFee": "number",
        "serviceFee": "number",
        "total": "number"
      },
      "paymentIntent": {
        "clientSecret": "string"
      }
    }
    ```
  - Error (400 Bad Request):
    ```json
    {
      "error": "Property not available for selected dates",
      "unavailableDates": ["YYYY-MM-DD"]
    }
    ```

#### Confirm Booking Payment
- **Endpoint**: `POST /api/bookings/{bookingId}/payment`
- **Description**: Confirm payment for a booking
- **Authentication**: Required (Booking owner only)
- **Request Body**:
  ```json
  {
    "paymentIntentId": "string",
    "paymentMethod": "string"
  }
  ```
- **Response**:
  - Success (200 OK):
    ```json
    {
      "bookingId": "integer",
      "status": "confirmed",
      "message": "Booking confirmed successfully"
    }
    ```

#### Get Booking Details
- **Endpoint**: `GET /api/bookings/{bookingId}`
- **Description**: Get details of a specific booking
- **Authentication**: Required (Booking guest, property host, or admin)
- **Response**:
  - Success (200 OK):
    ```json
    {
      "bookingId": "integer",
      "property": {
        "propertyId": "integer",
        "title": "string",
        "address": "string",
        "city": "string",
        "primaryImage": "string (URL)"
      },
      "host": {
        "hostId": "integer",
        "name": "string",
        "profileImage": "string (URL)"
      },
      "guest": {
        "guestId": "integer",
        "name": "string",
        "profileImage": "string (URL)"
      },
      "checkInDate": "YYYY-MM-DD",
      "checkOutDate": "YYYY-MM-DD",
      "numberOfGuests": "integer",
      "priceDetails": {
        "basePrice": "number",
        "cleaningFee": "number",
        "serviceFee": "number",
        "totalPrice": "number"
      },
      "status": "string",
      "paymentStatus": "string",
      "createdAt": "date"
    }
    ```

#### Get User Bookings
- **Endpoint**: `GET /api/users/bookings`
- **Description**: Get all bookings for the authenticated user
- **Authentication**: Required
- **Query Parameters**:
  - `role`: "guest" or "host" (required)
  - `status`: Filter by status (optional)
  - `page`: Page number (optional, default: 1)
  - `limit`: Items per page (optional, default: 10)
- **Response**:
  - Success (200 OK):
    ```json
    {
      "bookings": [
        {
          "bookingId": "integer",
          "propertyTitle": "string",
          "propertyImage": "string (URL)",
          "checkInDate": "YYYY-MM-DD",
          "checkOutDate": "YYYY-MM-DD",
          "totalPrice": "number",
          "status": "string",
          "otherParty": {
            "userId": "integer",
            "name": "string",
            "profileImage": "string (URL)"
          }
        }
      ],
      "pagination": {
        "currentPage": "integer",
        "totalPages": "integer",
        "totalItems": "integer"
      }
    }
    ```

#### Cancel Booking
- **Endpoint**: `POST /api/bookings/{bookingId}/cancel`
- **Description**: Cancel a booking
- **Authentication**: Required (Booking guest, property host, or admin)
- **Request Body**:
  ```json
  {
    "reason": "string"
  }
  ```
- **Response**:
  - Success (200 OK):
    ```json
    {
      "bookingId": "integer",
      "status": "cancelled_by_guest|cancelled_by_host",
      "refundAmount": "number (if applicable)",
      "message": "Booking cancelled successfully"
    }
    ```

#### Update Booking
- **Endpoint**: `PUT /api/bookings/{bookingId}`
- **Description**: Update booking details (dates or number of guests)
- **Authentication**: Required (Booking guest only)
- **Request Body**:
  ```json
  {
    "checkInDate": "YYYY-MM-DD (optional)",
    "checkOutDate": "YYYY-MM-DD (optional)",
    "numberOfGuests": "integer (optional)"
  }
  ```
- **Response**:
  - Success (200 OK):
    ```json
    {
      "bookingId": "integer",
      "status": "updated",
      "priceDetails": {
        "baseTotal": "number",
        "cleaningFee": "number",
        "serviceFee": "number",
        "total": "number",
        "priceDifference": "number"
      },
      "message": "Booking updated successfully"
    }
    ```
  - Error (400 Bad Request):
    ```json
    {
      "error": "Property not available for selected dates",
      "unavailableDates": ["YYYY-MM-DD"]
    }
    ```

### Validation Rules

#### Booking Creation/Update
- Check-in Date:
  - Required
  - Must be a future date (at least tomorrow)
  - Must be available in the property's calendar
- Check-out Date:
  - Required
  - Must be after check-in date
  - Maximum stay: 90 days
- Number of Guests:
  - Required
  - Must not exceed the property's maximum capacity
- Double Booking Prevention:
  - System must prevent booking dates that are already booked

### Business Rules
1. Cancellation Policy:
   - If cancelled by guest more than 7 days before check-in: 100% refund minus service fee
   - If cancelled by guest within 7 days of check-in: 50% refund minus service fee
   - If cancelled by host: 100% refund plus 10% compensation
2. Booking Status Flow:
   - pending → confirmed → completed
   - pending → declined
   - pending/confirmed → cancelled_by_guest/cancelled_by_host
3. Payment Processing:
   - Payment is authorized at booking creation
   - Payment is captured once host confirms booking
   - Refunds are processed based on cancellation policy

### Security Requirements
1. Only the booking guest, property host, or admin can view booking details
2. Payment information must be handled securely using payment gateway tokens
3. Date validation to prevent date manipulation attacks

### Performance Criteria
1. Availability checks should return results in under 300ms
2. Booking creation should complete within 2 seconds
3. System should handle up to 100 concurrent booking requests
4. Implement caching for availability data with a 1-minute TTL