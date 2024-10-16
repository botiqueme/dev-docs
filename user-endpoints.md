# User Management 
**Description**: This group of endpoints manages user accounts, including registration, login, ~~profile updates, and account deletion~~. Authentication tokens are required for most operations except for user registration, login, email confirmation, and password-related actions. All responses follow a standard format for consistency.

**DNS**: The DNS that will handle all the users-related backend operation is at __https://user.alfredhospitalityai.com/api/v1/__

## Endpoint Response Standard
All endpoints respond using the following standardized structure:

- **Success Response**:
```json
{
    "status": "success",
    "code": 200
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

### 1. GET /hello
- **Description**: Test the server availability.
- **Authentication**: No authentication required.
- **Example Request**:
```bash
curl https://user.alfredhospitalityai.com/api/v1/hello
```
- **Responses**:
  - **200 Success**
    ```json
    {
        "status": "success",
        "code": 200,
        "data": {
            "message": "Server is Online!"
        }
    }

### 1. POST /register
- **Description**: Registers a new account with the platform.
- **Authentication**: No authentication required.
- **Parameters**:
  - `email` (String, Required): Host’s email (e.g., `example@example.com`).
  - `password` (String, Required): Host’s password (minimum 8 characters, mix of letters and numbers).
- **Example Request**:
```bash
curl -X POST https://user.alfredhospitalityai.com/api/v1/register \
-H "Content-Type: application/json" \
-d '{
      "email": "john.doe@example.com",
      "password": "SecurePass123",
    }'
```
- **Responses**:
  - **201 Created**
    ```json
    {
        "status": "success",
        "data": {
            "message": "User registered successfully.",
        }
    }
    ```
    - **500 or 501 Generic Error**
    ```json
    {
        "status": "error",
        "code": 500 or 501,
        "message": "Generic error in user creation, contact the admin"
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
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Error in mail sending process"
    }
    ```
    
### 3. GET /verify/<token>
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
        "message": "Expired token."
    }
    ```
  - **401 Bad Request**
    ```json
    {
        "status": "error",
        "code": 401,
        "message": "Invalid token."
    }
    ```

### 2. POST /login
- **Description**: Authenticates a user and returns a session token.
- **Authentication**: No authentication required.
- **Parameters**:
  - `email` (String, Required): Host’s email (e.g., `example@example.com`).
  - `password` (String, Required): Host’s password.
- **Example Request**:
```bash
curl -X POST https://user.alfredhospitalityai.com/api/v1/login \
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
        "code": 200,
        "data": {
            "message": "successfull login"
            "jwt_token": jwt_token,
            "email": email
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
  - **402 mail not verified**
    ```json
    {
        "status": "error",
        "code": 402,
        "message": "Email not verified, yet"
    }
    ```



### 4. POST /users/forgot-password [NOT AVAILABLE]
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

### 5. PUT /users/reset-password [NOT AVAILABLE]
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

### 6. GET /users/profile [NOT AVAILABLE]
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

### 7. PUT /users/profile [NOT AVAILABLE]
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

### 8. DELETE /users/{`userId`} [NOT AVAILABLE]
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

### 9. GET /users/validate-token [NOT AVAILABLE]
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

### 10. POST /users/upload-avatar [NOT AVAILABLE]
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
