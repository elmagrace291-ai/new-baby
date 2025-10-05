# SmartBite Authentication & Role-Based Access Implementation

## Overview
This document explains the complete authentication and authorization system connecting the SmartBite frontend (React) with the ServeSoft backend (PHP/MySQL).

## Database Schema
The system uses a hierarchical role structure based on your existing database schema:

### Core Tables
- **User**: Base identity table (UserID, Name, Email, PhoneNumber)
- **Account**: Authentication credentials (AccountID, UserID, Password)
- **Customer**: Customer role (CustomerID, UserID)
- **Admin**: System administrator role (AdminID, UserID)
- **RestaurantStaff**: Base staff table (StaffID, UserID, RestaurantID, Role, Status)
- **RestaurantManager**: Manager role (ManagerID, StaffID)
- **DeliveryAgent**: Delivery driver role (DeliveryAgentID, StaffID)

## Supported Roles

### 1. Customer
- **Frontend**: `role: 'customer'`
- **Backend**: Customer table entry
- **Capabilities**: Browse menus, place orders, track deliveries
- **Registration**: Default role, always created

### 2. Restaurant Manager
- **Frontend**: `role: 'manager'`
- **Backend**: RestaurantStaff + RestaurantManager tables
- **Capabilities**: Manage restaurant, update menu, process orders
- **Registration**: Creates staff record with MANAGER role + manager entry + customer entry

### 3. Delivery Driver
- **Frontend**: `role: 'driver'`
- **Backend**: RestaurantStaff + DeliveryAgent tables
- **Capabilities**: View available deliveries, accept jobs, update delivery status
- **Registration**: Creates staff record with DELIVERY role + delivery agent entry + customer entry

### 4. System Admin
- **Frontend**: `role: 'admin'`
- **Backend**: Admin table entry
- **Capabilities**: Manage restaurants, users, and platform settings
- **Registration**: Only ONE admin allowed per system, validated during registration

## Implementation Details

### Backend Changes (api_auth.php)

#### Registration Endpoint
```php
POST /api_auth.php?action=register
```

**Request Body:**
```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "phone": "+237 680 123 456",
  "password": "SecurePass123!",
  "confirm": "SecurePass123!",
  "role": "customer|manager|driver|admin"
}
```

**Password Requirements:**
- Minimum 8 characters
- At least 1 uppercase letter
- At least 1 lowercase letter
- At least 1 number
- At least 1 special character

**Role-Specific Registration Logic:**
1. **Customer**: Creates User + Account + Customer
2. **Manager**: Creates User + Account + RestaurantStaff (MANAGER) + RestaurantManager + Customer
3. **Driver**: Creates User + Account + RestaurantStaff (DELIVERY) + DeliveryAgent + Customer
4. **Admin**: Creates User + Account + Admin (only if no admin exists)

**Response:**
```json
{
  "success": true,
  "message": "Account created successfully",
  "userId": 123
}
```

#### Login Endpoint
```php
POST /api_auth.php?action=login
```

**Request Body:**
```json
{
  "email": "john@example.com",
  "password": "SecurePass123!"
}
```

**Response:**
```json
{
  "success": true,
  "user": {
    "id": "u123",
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "+237 680 123 456",
    "role": "manager",
    "roleData": {
      "type": "manager",
      "id": 5,
      "staffId": 10,
      "restaurantId": 2
    }
  }
}
```

#### Check Admin Endpoint
```php
GET /api_auth.php?action=check_admin
```

**Response:**
```json
{
  "adminExists": true
}
```

### Frontend Changes

#### Register Component Updates
- Changed role options from `['customer', 'owner', 'agent']` to `['customer', 'manager', 'driver', 'admin']`
- Updated role display labels (manager → "Restaurant Manager", driver → "Delivery Agent")
- Admin role only shows if no admin exists (checked via API)
- Sends role data to backend during registration

#### API Service Updates
- Added `role` parameter to registration request
- Fixed `checkAdmin` endpoint to use `action=check_admin`
- Maintains session-based authentication

#### Auth Context Updates
- Updated User interface type definition to support new role names
- Removed deprecated 'owner' and 'agent' roles

### Configuration
**Environment Variables (.env):**
```
VITE_API_URL=http://localhost/github%20REpo/servesoft
```

This ensures the frontend can communicate with the PHP backend.

## User Flow Diagrams

### Registration Flow
1. User selects role on registration page
2. If admin role: Frontend checks if admin exists
3. User fills in required fields (name, email, password)
4. Frontend validates password strength
5. Frontend sends registration request with role
6. Backend validates and creates appropriate database records
7. Backend returns success
8. Frontend automatically logs user in
9. User redirected to appropriate dashboard

### Login Flow
1. User enters email and password
2. Backend verifies credentials
3. Backend fetches all user roles
4. Backend determines primary role (priority: admin > manager > driver > customer)
5. Backend creates session and returns user data
6. Frontend stores user in localStorage
7. Frontend redirects to role-specific dashboard

### Role Determination Priority
When a user has multiple roles:
1. Admin (highest priority)
2. Manager
3. Driver
4. Customer (default/lowest)

## Testing the Implementation

### Test Customer Registration
```
Name: Test Customer
Email: customer@test.com
Password: TestPass123!
Role: Customer
```

### Test Manager Registration
```
Name: Test Manager
Email: manager@test.com
Password: TestPass123!
Phone: +237 680 111 222
Role: Restaurant Manager
```

### Test Driver Registration
```
Name: Test Driver
Email: driver@test.com
Password: TestPass123!
Phone: +237 680 333 444
Role: Delivery Agent
```

### Test Admin Registration (First Time Only)
```
Name: System Admin
Email: admin@test.com
Password: TestPass123!
Role: Admin
```

## Security Features

1. **Password Hashing**: PHP `password_hash()` with bcrypt
2. **SQL Injection Protection**: Prepared statements throughout
3. **CORS Protection**: Configured headers in api_auth.php
4. **Session Management**: PHP sessions with credentials
5. **Input Validation**: Server-side validation for all fields
6. **Role Validation**: Ensures only valid roles accepted
7. **Admin Restriction**: Only one admin account allowed

## Database Transactions
All registration operations use database transactions to ensure:
- Atomic operations (all-or-nothing)
- Data consistency
- Automatic rollback on errors

## Error Handling

### Common Error Responses
- `400`: Invalid input (missing fields, weak password, invalid role)
- `401`: Invalid credentials (login)
- `409`: Email already exists
- `500`: Server error

### Example Error Response
```json
{
  "error": "Password must be at least 8 characters with uppercase, lowercase, number, and symbol"
}
```

## API Endpoints Summary

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api_auth.php?action=register` | POST | Create new user account |
| `/api_auth.php?action=login` | POST | Authenticate user |
| `/api_auth.php?action=check` | GET | Verify session |
| `/api_auth.php?action=logout` | GET | End session |
| `/api_auth.php?action=check_admin` | GET | Check if admin exists |

## Next Steps

### For Development
1. Test all role registrations
2. Verify role-based dashboard routing
3. Test multi-role scenarios
4. Implement password reset functionality
5. Add email verification (optional)

### For Production
1. Configure production API URL
2. Enable HTTPS
3. Set secure session cookies
4. Implement rate limiting
5. Add logging and monitoring
6. Configure backup strategy

## Troubleshooting

### Issue: Cannot register
- Verify database connection in config.php
- Check all required tables exist
- Ensure password meets requirements
- Verify email is not already in use

### Issue: Cannot login
- Check credentials are correct
- Verify user role records exist in database
- Check session configuration
- Verify CORS headers

### Issue: Admin option not showing
- Clear browser cache
- Check API endpoint `/api_auth.php?action=check_admin`
- Verify admin exists in database

### Issue: Wrong dashboard after login
- Check role determination logic in session_helper.php
- Verify roleData is returned correctly
- Check frontend routing configuration

## Contact
For issues or questions, contact: +237 680 938 302
