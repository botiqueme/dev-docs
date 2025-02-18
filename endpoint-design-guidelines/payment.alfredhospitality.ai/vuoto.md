## **Endpoint: `/enterprise_contact_us`**

## **1. Details**
- **Endpoint**: `/enterprise_contact_us`
- **Method**: `POST`
- **Authentication**: jwt.
- **Purpose**: Allow registered users to log in to the platform using email and password.

---

## **2. Accepted Parameters**
The request body must be in JSON format:

| **Parameter**  | **Type**  | **Required** | **Description**                         |
|---------------|----------|--------------|-----------------------------------------|
| `email`       | String   | Yes          | The user's registered email.           |
| `password`    | String   | Yes          | The user's password.                   |

### **Example Request**
```
POST /login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "mypassword"
}
```

---

## **3. Validations**
1. **Email**:
   - Ensure it is present and in a valid format (e.g., `user@example.com`).
2. **Password**:
   - Ensure it is present.
3. **Account Status (`is_active`)**:
   - Ensure that the account is active before allowing login.
   - If the user is deactivated, return an appropriate error message.
4. **Rate Limiting**:
   - Limit the number of consecutive attempts to prevent brute force attacks.

Example using Flask-Limiter:
```
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@v1.route('/login', methods=['POST'])
@limiter.limit("5 per minute")
def login():
    # Login logic
    pass
```

---

## **4. Endpoint Logic**
1. **Retrieve User**:
   - Search the database for the user corresponding to the provided email.
   - Return an error if the user does not exist.

2. **Check User Status**:
   - Verify if the user's email is verified (`is_verified`).
   - Verify if the account is **active (`is_active`)**.
   - If `is_verified` is `False`, return a `403 Forbidden` error.
   - If `is_active` is `False`, return a `403 Forbidden` error with a message prompting reactivation.

3. **Verify Password**:
   - Use `bcrypt` to compare the provided password with the hashed one in the database.
   - Return an error if the password does not match.

4. **Generate JWT Token**:
   - Create an access token (`JWT`) with:
     - `user_id`: The user's unique ID.
     - `email`: The user's email.
     - `is_active`: The current user status.
     - `exp`: Token expiration (e.g., 1 hour).
   - Sign the token with the backend's secret key.

5. **Response to Frontend**:
   - Return the JWT token, `user_id`, `is_active` status, and a success message.

---

## **5. Security Measures**
1. **HTTPS**:
   - Mandatory for all requests to protect credentials and JWT tokens.

2. **JWT Token Security**:
   - Use a robust secret key (`SECRET_KEY`).
   - Configure an appropriate expiration (e.g., 1 hour).
   - Add `iat` (issued at) and `jti` (unique identifier) to the JWT payload (**optional improvement**).

3. **Token Blacklist (**optional improvement**)**:
   - Implement a blacklist to invalidate tokens in case of logout or compromise.

4. **Protection Against Brute Force**:
   - Implement rate limiting on the `/login` endpoint.

---

## **6. Endpoint Responses**
### **Success**
- **HTTP Status**: `200 OK`
- **Body**:
```
{
  "status": "success",
  "code": 200,
  "data": {
    "message": "Login successful",
    "jwt_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user_id": "123e4567-e89b-12d3-a456-426614174000",
    "is_active": true
  }
}
```

### **Errors**
#### **1. User Not Found**
- **HTTP Status**: `401 Unauthorized`
- **Body**:
```
{
  "status": "error",
  "code": 401,
  "message": "Incorrect email or password."
}
```

#### **2. Email Not Verified**
- **HTTP Status**: `403 Forbidden`
- **Body**:
```
{
  "status": "error",
  "code": 403,
  "message": "Email not verified."
}
```

#### **3. Account is Deactivated**
- **HTTP Status**: `403 Forbidden`
- **Body**:
```
{
  "status": "error",
  "code": 403,
  "message": "Account is deactivated. Please reactivate your account to proceed."
}
```

#### **4. Rate Limiting Exceeded**
- **HTTP Status**: `429 Too Many Requests`
- **Body**:
```
{
  "status": "error",
  "code": 429,
  "message": "Too many login attempts. Please try again later."
}
```

---

## **7. Implementation Code**
```
@v1.route('/login', methods=['POST'])
@limiter.limit("5 per minute")
def login():
    """
    Endpoint for user authentication.
    """

    data = request.get_json()
    email = data.get('email')
    password = data.get('password')

    # Input validation
    if not email or not password:
        return jsonify_return_error("error", 400, "Email and password are required."), 400

    # Retrieve user from database
    user = User.query.filter_by(email=email).first()
    if not user:
        return jsonify_return_error("error", 401, "Incorrect email or password."), 401

    # Check if user is verified
    if not user.is_verified:
        return jsonify_return_error("error", 403, "Email not verified."), 403

    # Check if user is active
    if not user.is_active:
        return jsonify_return_error("error", 403, "Account is deactivated. Please reactivate your account to proceed."), 403

    # Verify password
    if not verify_password(user.password_hash, password):
        return jsonify_return_error("error", 401, "Incorrect email or password."), 401

    # Generate JWT tokens
    access_token = create_jwt_token(user, token_type="access")
    refresh_token = create_jwt_token(user, token_type="refresh")

    # Return response
    return jsonify_return_success("success", 200, {
        "message": "Login successful",
        "jwt_token": access_token,
        "user_id": user.user_id,
        "is_active": user.is_active
    }), 200
```

---

## **8. Future Improvements**
1. **JWT Token Blacklist (**optional improvement**)**:
   - Implement a blacklist to invalidate JWT tokens upon logout or compromise.

2. **Temporary IP Blocking for Failed Attempts (**optional improvement**)**:
   - Block IPs temporarily after exceeding a certain number of failed login attempts.

3. **Advanced Monitoring (**optional improvement**)**:
   - Integrate monitoring tools like **Prometheus** or **Grafana** to gather detailed metrics:
     - Number of successful and failed logins.
     - Login attempts per IP.
     - Frequency of blocked accounts.

4. **Enhanced User Feedback (**optional improvement**)**:
   - Include the remaining number of attempts before blocking in error responses to improve user experience.

