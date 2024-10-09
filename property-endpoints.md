# Property Management 
**Description**: This group manages properties registered by hosts, including adding, updating, retrieving, and activating/deactivating properties. Authentication tokens are required for all operations to ensure only authenticated users can manage properties. All responses follow a standard structure for consistency.

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

### 1. POST /properties
- **Description**: Creates a new property for the host.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Parameters**:
  - `user_id` (UUID, Required): Host’s unique identifier.
  - `property_name` (String, Required): The name of the property.
  - `address` (String, Required): Physical address of the property.
  - `max_occupancy` (Integer, Required): Maximum number of guests the property can accommodate.
- **Example Request**:
  ```bash
  curl -X POST https://api.example.com/v1/properties \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
        "user_id": "uuid-of-user",
        "property_name": "Beach House",
        "address": "123 Ocean Drive",
        "max_occupancy": 8
      }'
  ```
- **Responses**:
  - **201 Created**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Property created successfully.",
            "property_id": "uuid-of-the-new-property"
        }
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Invalid input data - missing required fields or invalid format."
    }
    ```

### 2. GET /properties
- **Description**: Retrieves all properties associated with a particular host.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Parameters**:
  - `user_id` (UUID, Required): Host’s unique identifier.
- **Example Request**:
  ```bash
  curl -X GET https://api.example.com/v1/properties?user_id=uuid-of-user \
  -H "Authorization: Bearer <token>"
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": [
            {
                "property_id": "uuid-of-property-1",
                "property_name": "Property 1",
                "address": "123 Property St.",
                "max_occupancy": 4,
                "status": "active"
            },
            {
                "property_id": "uuid-of-property-2",
                "property_name": "Property 2",
                "address": "456 Another St.",
                "max_occupancy": 2,
                "status": "inactive"
            }
        ]
    }
    ```
  - **404 Not Found**
    ```json
    {
        "status": "error",
        "code": 404,
        "message": "No properties found for the user."
    }
    ```

### 3. GET /properties/{`propertyId`}
- **Description**: Retrieves details of a specific property.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Path Parameters**:
  - `propertyId` (UUID, Required): Unique identifier for the property.
- **Example Request**:
  ```bash
  curl -X GET https://api.example.com/v1/properties/uuid-of-property \
  -H "Authorization: Bearer <token>"
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "property_id": "uuid-of-property",
            "property_name": "Property 1",
            "address": "123 Property St.",
            "max_occupancy": 4,
            "status": "active"
        }
    }
    ```
  - **404 Not Found**
    ```json
    {
        "status": "error",
        "code": 404,
        "message": "Property not found."
    }
    ```

### 4. PUT /properties/{`propertyId`}
- **Description**: Updates information for a specified property.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Path Parameters**:
  - `propertyId` (UUID, Required): Unique identifier for the property.
- **Parameters**:
  - `property_name` (String, Optional): Updated name of the property.
  - `address` (String, Optional): Updated address of the property.
  - `max_occupancy` (Integer, Optional): Updated maximum occupancy.
- **Example Request**:
  ```bash
  curl -X PUT https://api.example.com/v1/properties/uuid-of-property \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
        "property_name": "Updated Beach House",
        "address": "789 Updated Drive",
        "max_occupancy": 10
      }'
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Property updated successfully."
        }
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Invalid input data."
    }
    ```

### 5. DELETE /properties/{`propertyId`}
- **Description**: Deletes a property from the system.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Path Parameters**:
  - `propertyId` (UUID, Required): Unique identifier for the property to be deleted.
- **Example Request**:
  ```bash
  curl -X DELETE https://api.example.com/v1/properties/uuid-of-property \
  -H "Authorization: Bearer <token>"
  ```
- **Responses**:
  - **204 No Content**
  - **404 Not Found**
    ```json
    {
        "status": "error",
        "code": 404,
        "message": "Property not found."
    }
    ```

### 6. POST /properties/{`propertyId`}/activate **(inferred)**
- **Description**: Activates a property for production use.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Path Parameters**:
  - `propertyId` (UUID, Required): Unique identifier for the property.
- **Example Request**:
  ```bash
  curl -X POST https://api.example.com/v1/properties/uuid-of-property/activate \
  -H "Authorization: Bearer <token>"
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Property activated successfully."
        }
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Activation failed due to missing or invalid data."
    }
    ```

### 7. POST /properties/{`propertyId`}/deactivate **(inferred)**
- **Description**: Deactivates a property temporarily, removing it from production.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Path Parameters**:
  - `propertyId` (UUID, Required): Unique identifier for the property.
- **Example Request**:
  ```bash
  curl -X POST https://api.example.com/v1/properties/uuid-of-property/deactivate \
  -H "Authorization: Bearer <token>"
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Property deactivated successfully."
        }
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Deactivation failed due to missing or invalid data."
    }
    ```