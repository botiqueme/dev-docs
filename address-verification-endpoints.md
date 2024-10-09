---
sidebar_label: Address verification
---
# Address Verification Integration (Google Maps)
**Description**: This group integrates with the Google Maps API to verify addresses for billing and property locations. Authentication tokens are required for all operations to ensure only authenticated users can request address verifications. All responses follow a standard structure for consistency.

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

### 1. POST /verification/address
- **Description**: Validates an address through the Google Maps API to ensure it is accurate and formatted correctly for billing or property registration purposes.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Parameters**:
  - `user_id` (UUID, Required): Host’s unique identifier.
  - `address` (String, Required): The address to be validated.
- **Example Request**:
  ```bash
  curl -X POST https://api.example.com/v1/verification/address \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
        "user_id": "uuid-of-user",
        "address": "123 Main St, City, Country"
      }'
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Address validated successfully.",
            "formatted_address": "123 Valid St, City, Country",
            "latitude": 40.712776,
            "longitude": -74.005974
        }
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Invalid address format or missing required fields."
    }
    ```
  - **404 Not Found**
    ```json
    {
        "status": "error",
        "code": 404,
        "message": "Address could not be verified."
    }
    ```

### 2. GET /verification/status **(inferred)**
- **Description**: Retrieves the status of a previously requested address verification. This endpoint checks the outcome of the address validation process initiated earlier.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Parameters**:
  - `verification_id` (UUID, Required): Unique identifier for the address verification request.
- **Example Request**:
  ```bash
  curl -X GET https://api.example.com/v1/verification/status?verification_id=uuid-of-verification \
  -H "Authorization: Bearer <token>"
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "verification_id": "uuid-of-the-verification",
            "status": "completed",
            "result": {
                "is_valid": true,
                "formatted_address": "123 Valid St, City, Country",
                "coordinates": {
                    "latitude": 40.712776,
                    "longitude": -74.005974
                }
            }
        }
    }
    ```
  - **404 Not Found**
    ```json
    {
        "status": "error",
        "code": 404,
        "message": "Verification request not found."
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Invalid verification ID."
    }
    ```