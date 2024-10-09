# System and Testing 
**Description**: This group provides endpoints for system health checks, version retrieval, deployment management, and testing functionalities. Authentication tokens are required for most operations to ensure they are only accessible to authorized users. All responses follow a standard structure for consistency.

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

### 1. GET /health-check
- **Description**: Checks the health status of the system to ensure the backend is operational.
- **Authentication**: No authentication required.
- **Parameters**: None
- **Example Request**:
  ```bash
  curl -X GET https://api.example.com/v1/health-check
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "status": "healthy",
            "uptime": "72 hours"
        }
    }
    ```

### 2. GET /version
- **Description**: Retrieves the current version of the backend API.
- **Authentication**: No authentication required.
- **Parameters**: None
- **Example Request**:
  ```bash
  curl -X GET https://api.example.com/v1/version
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "version": "v1.3.0",
            "release_date": "2024-10-01"
        }
    }
    ```

### 3. POST /deploy/staging
- **Description**: Deploys a new version of the platform to the staging environment.
- **Authentication**: Requires a valid deployment token (`Authorization: Bearer <token>`).
- **Parameters**:
  - `version` (String, Required): Version number to deploy.
  - `changelog` (Text, Optional): Changelog information for the deployment.
- **Example Request**:
  ```bash
  curl -X POST https://api.example.com/v1/deploy/staging \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
        "version": "v1.3.0",
        "changelog": "Bug fixes and performance improvements"
      }'
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Deployment to staging environment triggered successfully.",
            "version": "v1.3.0"
        }
    }
    ```
  - **401 Unauthorized**
    ```json
    {
        "status": "error",
        "code": 401,
        "message": "Invalid deployment token."
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Missing required parameters."
    }
    ```

### 4. POST /deploy/production
- **Description**: Deploys a new version of the platform to the production environment.
- **Authentication**: Requires a valid deployment token (`Authorization: Bearer <token>`).
- **Parameters**:
  - `version` (String, Required): Version number to deploy.
  - `authorization` (String, Required): Token for authorizing the deployment.
- **Example Request**:
  ```bash
  curl -X POST https://api.example.com/v1/deploy/production \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
        "version": "v1.3.0",
        "authorization": "deployment-auth-token"
      }'
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Production deployment triggered successfully.",
            "version": "v1.3.0"
        }
    }
    ```
  - **403 Forbidden**
    ```json
    {
        "status": "error",
        "code": 403,
        "message": "Unauthorized access - invalid authorization token."
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Invalid or missing parameters."
    }
    ```

### 5. POST /testing/clear-database **(inferred)**
- **Description**: Clears the database in the staging environment for testing purposes. This action is irreversible and should be restricted to authorized users.
- **Authentication**: Requires a valid session token with elevated permissions (`Authorization: Bearer <token>`).
- **Parameters**:
  - `auth_token` (String, Required): Security token for clearing the test database.
- **Example Request**:
  ```bash
  curl -X POST https://api.example.com/v1/testing/clear-database \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
        "auth_token": "secure-auth-token"
      }'
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Test database cleared successfully."
        }
    }
    ```
  - **401 Unauthorized**
    ```json
    {
        "status": "error",
        "code": 401,
        "message": "Invalid security token."
    }
    ```

### 6. GET /testing/test-data **(inferred)**
- **Description**: Retrieves test data for various test scenarios. This endpoint is used to fetch predefined datasets for testing system features and functionality.
- **Authentication**: Requires a valid session token with elevated permissions (`Authorization: Bearer <token>`).
- **Parameters**:
  - `test_type` (String, Optional): Specifies the type of test data to retrieve (e.g., `"properties"`, `"users"`).
- **Example Request**:
  ```bash
  curl -X GET https://api.example.com/v1/testing/test-data?test_type=properties \
  -H "Authorization: Bearer <token>"
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "test_type": "properties",
            "data": [
                {
                    "property_id": "uuid-of-property-1",
                    "property_name": "Test Property 1",
                    "address": "123 Test St.",
                    "max_occupancy": 4
                },
                {
                    "property_id": "uuid-of-property-2",
                    "property_name": "Test Property 2",
                    "address": "456 Test Ave.",
                    "max_occupancy": 2
                }
            ]
        }
    }
    ```
  - **404 Not Found**
    ```json
    {
        "status": "error",
        "code": 404,
        "message": "Test type not found."
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Invalid or missing parameters."
    }
    ```