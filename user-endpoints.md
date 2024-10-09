# User Management 
**Description**: This group of endpoints manages user accounts, including registration, login, profile updates, and account deletion. Authentication tokens are required for most operations except for user registration, login, email confirmation, and password-related actions. All responses follow a standard format for consistency.

## Endpoint Response Standard
All endpoints respond using the following standardized structure:

- **Success Response**:
```json
{
    "status": "success",
    "data": { /* response object */ }
}
```

- **Error Response**:
```json
{
    "status": "error",
    "code": 400,
    "message": "Detailed error message"
}
```

## Endpoints:

### 1. POST /users/register
- **Description**: Registers a new host account with the platform.
- **Authentication**: No authentication required.
- **Parameters**:
  - `name` (String, Required): Host’s first name.
  - `surname` (String, Required): Host’s surname.
  - `email` (String, Required): Host’s email (e.g., `example@example.com`).
  - `password` (String, Required): Host’s password (minimum 8 characters, mix of letters and numbers).
  - `billing_address` (String, Required): Host’s billing address.
  - `vat_number` (String, Optional): VAT number for billing.
- **Example Request**:
```bash
curl -X POST https://api.example.com/v1/users/register \
-H "Content-Type: application/json" \
-d '{
      "name": "John",
      "surname": "Doe",
      "email": "john.doe@example.com",
      "password": "SecurePass123",
      "billing_address": "123 Billing St.",
      "vat_number": "VAT12345"
    }'
```
- **Responses**:
  - **201 Created**
    ```json
    {
        "status": "success",
        "data": {
            "message": "User registered successfully.",
            "user_id": "uuid-of-the-new-user"  // Note: This is an example representation.
        }
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Validation error - invalid email format or missing required fields."
    }
    ```
  - **409 Conflict**
    ```json
    {
        "status": "error",
        "code": 409,
        "message": "Email already registered."
    }
    ```

### 2. POST /users/login
- **Description**: Authenticates a user and returns a session token.
- **Authentication**: No authentication required.
- **Parameters**:
  - `email` (String, Required): Host’s email (e.g., `example@example.com`).
  - `password` (String, Required): Host’s password.
- **Example Request**:
```bash
curl -X POST https://api.example.com/v1/users/login \
-H "Content-Type: application/json" \
-d '{
      "email": "john.doe@example.com",
      "password": "SecurePass123"
    }'
```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "token": "session-token",
            "user_id": "uuid-of-the-user"  // Note: This is an example representation.
        }
    }
    ```
  - **401 Unauthorized**
    ```json
    {
        "status": "error",
        "code": 401,
        "message": "Incorrect email or password."
    }
    ```

### 3. POST /users/confirm-email
- **Description**: Confirms the host’s email using a token sent during registration.
- **Authentication**: No authentication required.
- **Parameters**:
  - `token` (String, Required): Email verification token sent to the user’s email.
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Email confirmed successfully."
        }
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Invalid or expired token."
    }
    ```

### 4. POST /users/forgot-password
- **Description**: Initiates the password recovery process for a user.
- **Authentication**: No authentication required.
- **Parameters**:
  - `email` (String, Required): Host’s email.
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Password reset link sent to the provided email."
        }
    }
    ```
  - **404 Not Found**
    ```json
    {
        "status": "error",
        "code": 404,
        "message": "Email not found."
    }
    ```

### 5. PUT /users/reset-password
- **Description**: Resets the user’s password using a token.
- **Authentication**: No authentication required.
- **Parameters**:
  - `token` (String, Required): Password reset token.
  - `new_password` (String, Required): New password for the account (minimum 8 characters).
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Password reset successfully."
        }
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Invalid token or password criteria not met."
    }
    ```

### 6. GET /users/profile
- **Description**: Retrieves the profile details of the logged-in host.
- **Authentication**: Requires a valid session token.
- **Parameters**:
  - `user_id` (UUID, Required): Unique identifier for the logged-in host.
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "user_id": "uuid-of-the-user",  // Note: This is an example representation.
            "name": "Host Name",
            "surname": "Host Surname",
            "email": "host@example.com",
            "billing_address": "123 Billing St.",
            "vat_number": "VAT123456"
        }
    }
    ```
  - **404 Not Found**
    ```json
    {
        "status": "error",
        "code": 404,
        "message": "User not found."
    }
    ```

### 7. PUT /users/profile
- **Description**: Updates the user’s profile information, such as name or billing address.
- **Authentication**: Requires a valid session token.
- **Parameters**:
  - `user_id` (UUID, Required): Host’s unique identifier.
  - `name` (String, Optional): Updated name.
  - `surname` (String, Optional): Updated surname.
  - `billing_address` (String, Optional): Updated billing address.
  - `phone_number` (String, Optional): Host’s phone number (format: `+country_code-number`).
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Profile updated successfully."
        }
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Invalid format for one or more fields."
    }
    ```

### 8. DELETE /users/{`userId`}
- **Description**: Deletes a user account from the system.
- **Authentication**: Requires a valid session token.
- **Path Parameters**:
  - `userId` (UUID, Required): Unique identifier for the host to be deleted.
- **Responses**:
  - **204 No Content**
  - **404 Not Found**
    ```json
    {
        "status": "error",
        "code": 404,
        "message": "User ID not found."
    }
    ```

### 9. GET /users/validate-token
- **Description**: Validates the user’s session token for authentication purposes.
- **Authentication**: Requires a valid session token.
- **Parameters**:
  - `token` (String, Required): Authentication token.
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Token is valid."
        }
    }
    ```
  - **401 Unauthorized**
    ```json
    {
        "status": "error",
        "code": 401,
        "message": "Token is invalid or expired."
    }
    ```

### 10. POST /users/upload-avatar
- **Description**: Uploads or updates the user’s avatar image.
- **Authentication**: Requires a valid session token.
- **Parameters**:
  - `user_id` (UUID, Required): Host’s unique identifier.
  - `avatar` (File, Required): Image file (JPEG or PNG).
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Avatar uploaded successfully."
        }
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Invalid file format."
    }
    ```
