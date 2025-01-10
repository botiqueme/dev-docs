# Endpoint: `/login`

## 1. Details
- **Endpoint**: `/login`
- **Method**: `POST`
- **Authentication**: None (initial authentication endpoint)..
- **Purpose**: Allow registered users to log in to the platform using email and password.

---

## 2. Accepted Parameters
The request body must be in JSON format:
- `email` (string, required): The user's registered email.
- `password` (string, required): The user's password.

Example request:
```
{
  "email": "user@example.com",
  "password": "mypassword"
}
```

---

## 3. Validations
1. **Email**:
   - Ensure it is present and in a valid format (e.g., user@example.com) (Handled by the frontend).
2. **Password**:
   - Ensure it is present.
3. **Rate Limiting**:
   - Limit the number of consecutive attempts to prevent brute force attacks.

Example using Flask-Limiter:
```
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@v1.route('/login', methods=['POST'])
@limiter.limit("5 per minute")
def login():
    # Logica del login
    pass
```

---

## 4. Endpoint Logic
1. **User Retrieval:**:
   - Search the database for the user corresponding to the provided email.
   - Return an error if the user does not exist.

2. **User Status Verification:**:
   - Check if the user has verified their email (is_verified).
   - Return an error if the email is not verified.

3. **Password Verification:**:
   - Use bcrypt to compare the provided password with the hashed one in the database.
   - Return an error if the password does not match.

4. **JWT Token Generation:**:
   - Create a JWT token with the following information:
     - `user_id`: The user's unique ID.
     - `email`: The user's email.
     - `exp`: Token expiration (e.g., 1 hour).
   - Sign the token with the backend's secret key.

5. **Response to Frontend:**:
   - Return the JWT token, user_id, and a success message.

---

## 5. Sicurezza
1. **HTTPS**:
   - Mandatory for all requests to protect credentials and JWT tokens.

2. **JWT Token Security:**:
   - Use a robust secret key (SECRET_KEY).
   - Configure an appropriate expiration (e.g., 1 hour).
   - Add iat (issued at) and jti (unique identifier) to the JWT payload (optional improvement) for enhanced security..

3. **Token Blacklist (optional improvement):**:
   - Implement a blacklist to invalidate tokens in case of logout or compromise.
   - 
4. **Protection Against Brute Force:**:
   - Implement rate limiting on the /login endpoint.

---

## 6. Endpoint Responses

### Success
- **HTTP Status**: `200 OK`
- **Body**:
```
{
  "status": "success",
  "code": 200,
  "data": {
    "message": "Login successful",
    "jwt_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user_id": "123e4567-e89b-12d3-a456-426614174000"
  }
}
```

### Errors

1. **User Not Found**:
   - **HTTP Status**: `401 Unauthorized`
   - **Body**:
```
{
  "status": "error",
  "code": 401,
  "message": "Incorrect email or password."
}
```

2. **Email not verified**:
   - **HTTP Status**: `403 Forbidden`
   - **Body**:
```
{
  "status": "error",
  "code": 403,
  "message": "Email not verified."
}
```

3. **Rate Limiting**:
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

## 7. Code

```
@v1.route('/login', methods=['POST'])
@limiter.limit("5 per minute")
def login():
    """
    Endpoint per effettuare il login dell'utente.
    """

    data = request.get_json()
    email = data.get('email')
    password = data.get('password')

    # Validazione dell'input
    if not email or not password:
        return utils.jsonify_return_error("error", 400, "Email and password are required."), 400


    # Cerca l'utente nel database
    user = User.query.filter_by(email=email).first()
    if user is None:
        return utils.jsonify_return_error("error", 401, "Incorrect email or password."), 401
    if not user.is_verified:
        return utils.jsonify_return_error("error", 403, "Email not verified, yet"), 403


    # Verifica la password utilizzando bcrypt
    if not utils.verify_password(user.password_hash, password):  # Confronto con la password hashata
        return utils.jsonify_return_error("error", 401, "Incorrect email or password."), 401


    # Generazione dei Token JWT
    access_token = utils.create_jwt_token(user, token_type="access")
    refresh_token = utils.create_jwt_token(user, token_type="refresh")

    # Risposta al Frontend
    data = {
        "message": "Login successful",
        "access_jwt_token": access_token,
        "refresh_jwt_token": refresh_token,
        "user_id": user.id,
        "email": user.email
    }

    response = utils.jsonify_return_success("success", 200, data)
    return response, 200

```

---

## 8. Future Improvements for the /login Endpoint

1. **JWT Token Blacklist (optional improvement):**:
   - Implement a blacklist to invalidate JWT tokens in case of logout or compromise.

2. **IP Blocking for Failed Attempts (optional improvement):**:
   - Configure a mechanism to temporarily block IPs exceeding a certain number of failed login attempts.

3. **Advanced Monitoring (optional improvement):**:
   - Integrate monitoring tools like Prometheus or Grafana to gather detailed metrics:
     - Number of successful and failed logins.
     - Login attempts per IP and frequency of blocks.

4. **Enhanced Feedback (optional improvement):**:
   - Include the remaining number of attempts before blocking in the responses to improve user experience.

---

