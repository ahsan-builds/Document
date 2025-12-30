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


---

## API Philosophy and Design Standards

### REST Conventions
- **Resource-oriented endpoints** with nouns, not verbs.
- **Stateless requests** with explicit inputs and authentication context.
- **Consistent error envelopes** that include error code, message, and field-level details.

### Validation and Serialization
- Centralized **serializer validation** ensures a single source of truth.
- **Explicit field constraints** prevent ambiguous behaviors (e.g., email formatting, password strength).

---

## Authentication and Session Management (Deep Dive)

### Background Theory
Session-based authentication uses a server-side store keyed by a secure cookie (`sessionid`). This supports server-driven invalidation and built-in CSRF protection.

### Session Lifecycle
1. User submits credentials to `/users/login/`.
2. Backend validates credentials and issues a session cookie.
3. Cookie is sent on subsequent requests by the browser.
4. Logout invalidates server-side session state.

### Practical Example: Login Sequence
```http
POST /users/login/
Content-Type: application/json
{
  "email": "recruiter@acme.com",
  "password": "••••••••"
}
```
**Expected Response**: `200 OK` + `Set-Cookie: sessionid=...; HttpOnly`

---

## Use Cases

### Use Case: Secure Administrator Onboarding
- **Actors**: HR Admin, Auth service
- **Goal**: Register a new admin and enforce password policy
- **Flow**: Register → Login → Change password on first login
- **Outcome**: Admin is authenticated with audited login history

### Use Case: Account Recovery (Future)
- **Goal**: Enable password reset without exposing account enumeration
- **Notes**: Would require a secure token flow and email verification.

---

## Best Practices for Consumers

- **Send credentials over HTTPS only**.
- **Use `withCredentials: true`** in browser requests to send cookies.
- **Retry only idempotent operations**; do not retry login without user confirmation.
- **Avoid storing session cookies in local storage**.

---

## Common Errors and Troubleshooting

| Error | Likely Cause | Resolution |
|------|--------------|-----------|
| 403 CSRF Failed | Missing CSRF token | Include CSRF token header in unsafe methods. |
| 401 Unauthorized | Session expired | Re-authenticate to obtain a new session. |
| 400 Bad Request | Validation error | Check serializer error details. |

---

## Limitations

- Session authentication is primarily optimized for browser clients.
- Horizontal scaling requires shared session storage (e.g., Redis) if default DB sessions are insufficient.

---

## FAQ

**Q: Can we use JWT alongside session auth?**  
A: Yes, but it requires separate authentication classes and clear client segmentation.

**Q: Why store login history?**  
A: It supports audit trails, anomaly detection, and compliance.

---

## Future Enhancements

- **Multi-factor authentication (MFA)** for privileged accounts.
- **Session management UI** for admins (active sessions list, remote logout).
- **Password rotation policies** configurable per organization.
