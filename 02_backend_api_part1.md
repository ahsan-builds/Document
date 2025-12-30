# Hiro Backend - Complete API Documentation

## API Base URL
**Development**: `http://127.0.0.1:8000`

---

## 1. USER AUTHENTICATION & MANAGEMENT

### 1.1 Register User

**Endpoint**: `POST /users/register/`

**Handler**: `server/users/views.py::RegisterView.post()` (Line 40-56)

**Purpose**: Creates a new user account with profile information. Uses Django's built-in `User` model extended with a `UserProfile` one-to-one relationship.

**Request Headers**:
```
Content-Type: application/json
```

**Request Body (JSON)**:
```json
{
  "username": "string (required, unique)",
  "first_name": "string",
  "last_name": "string",
  "email": "string (valid email)",
  "password": "string (min 8 chars)",
  "password_confirm": "string (must match password)",
  "telephone": "string",
  "department": "string",
  "position": "string",
  "language": "string (default: 'en')",
  "timezone": "string (default: 'UTC')"
}
```

**Business Logic** (from `users/serializers.py::UserRegisterSerializer.create()`):
1. Validates password == password_confirm
2. Creates `User` instance using `User.objects.create_user()`
3. Hashes password using Django's `set_password()`
4. Creates related `UserProfile` with additional fields
5. Sets `password_last_changed` to current timestamp

**Response 201 Created**:
```json
{
  "id": 2,
  "username": "farhan1",
  "first_name": "Farhan",
  "last_name": "Dev",
  "email": "farhan@example.com",
  "profile": {
    "telephone": "03214567890",
    "avatar": null,
    "department": "Engineering",
    "position": "Backend Developer",
    "language": "en",
    "timezone": "Asia/Karachi",
    "password_last_changed": "2025-11-24T05:43:45.102880Z"
  }
}
```

**Database Tables Modified**:
- `auth_user`: Stores username, email, password (hashed)
- `users_userprofile`: Stores profile fields

---

### 1.2 Login

**Endpoint**: `POST /users/login/`

**Handler**: `server/users/views.py::LoginView.post()` (Line 69-97)

**Purpose**: Authenticates user credentials and creates a Django session. Sets a `sessionid` cookie for subsequent authenticated requests.

**Request Body (JSON)**:
```json
{
  "username": "farhan1",
  "password": "StrongPass123!"
}
```

**Business Logic**:
1. **Validation** (Line 74-75): Extracts username/password from serializer
2. **Authentication** (Line 77): Calls Django's `authenticate()` which:
   - Queries `User` table by username
   - Verifies password using `check_password()` (compares hashed password)
3. **Session Creation** (Line 85): Calls `login(request, user)` which:
   - Creates session in `django_session` table
   - Generates session key
   - Sets `sessionid` cookie in response
4. **Login History** (Line 87-92): Creates `LoginHistory` record with:
   - User agent from `request.META['HTTP_USER_AGENT']`
   - Timestamp (auto-generated)
   - Event type: "login"

**Response 200 OK**:
```json
{
  "id": 2,
  "username": "farhan1",
  "first_name": "Farhan",
  "last_name": "Dev",
  "email": "farhan@example.com",
  "profile": { ... }
}
```

**Response Headers**:
```
Set-Cookie: sessionid=<session_key>; HttpOnly; Path=/; SameSite=Lax
```

**Error 400 Bad Request**:
```json
{
  "detail": "Invalid username or password."
}
```

