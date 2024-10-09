# Property KB Management 
**Description**: This group manages the knowledge base associated with each property, allowing hosts to store and manage informational entries for their guests. Authentication tokens are required for all operations to ensure only authorized users can access and modify the knowledge base. All responses follow a standard structure for consistency.

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

### 1. POST /properties/{`propertyId`}/knowledge-base
- **Description**: Creates a new entry or updates an existing one in the knowledge base for a specific property.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Path Parameters**:
  - `propertyId` (UUID, Required): Unique identifier for the property.
- **Parameters**:
  - `entry_title` (String, Required): Title of the knowledge base entry.
  - `content` (Text, Required): Content details for the entry.
- **Example Request**:
  ```bash
  curl -X POST https://api.example.com/v1/properties/uuid-of-property/knowledge-base \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
        "entry_title": "Check-in Instructions",
        "content": "Guests should check in at the front desk with their ID."
      }'
  ```
- **Responses**:
  - **201 Created**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Knowledge base entry created successfully.",
            "entry_id": "uuid-of-the-entry"
        }
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Invalid content format or missing required fields."
    }
    ```

### 2. GET /properties/{`propertyId`}/knowledge-base
- **Description**: Retrieves all knowledge base entries for a specific property.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Path Parameters**:
  - `propertyId` (UUID, Required): Unique identifier for the property.
- **Example Request**:
  ```bash
  curl -X GET https://api.example.com/v1/properties/uuid-of-property/knowledge-base \
  -H "Authorization: Bearer <token>"
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": [
            {
                "entry_id": "uuid-of-entry-1",
                "entry_title": "Check-in Instructions",
                "content": "Guests should check in at the front desk with their ID."
            },
            {
                "entry_id": "uuid-of-entry-2",
                "entry_title": "Wi-Fi Information",
                "content": "Wi-Fi Name: PropertyWiFi, Password: 12345678"
            }
        ]
    }
    ```
  - **404 Not Found**
    ```json
    {
        "status": "error",
        "code": 404,
        "message": "No knowledge base entries found for this property."
    }
    ```

### 3. PUT /properties/{`propertyId`}/knowledge-base/{`entryId`}
- **Description**: Updates an existing knowledge base entry for a specific property.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Path Parameters**:
  - `propertyId` (UUID, Required): Unique identifier for the property.
  - `entryId` (UUID, Required): Unique identifier for the knowledge base entry.
- **Parameters**:
  - `content` (Text, Required): Updated content of the knowledge base entry.
- **Example Request**:
  ```bash
  curl -X PUT https://api.example.com/v1/properties/uuid-of-property/knowledge-base/uuid-of-entry \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
        "content": "Updated check-in instructions."
      }'
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Knowledge base entry updated successfully."
        }
    }
    ```
  - **404 Not Found**
    ```json
    {
        "status": "error",
        "code": 404,
        "message": "Knowledge base entry not found."
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

### 4. DELETE /properties/{`propertyId`}/knowledge-base/{`entryId`}
- **Description**: Deletes a specific knowledge base entry for a property.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Path Parameters**:
  - `propertyId` (UUID, Required): Unique identifier for the property.
  - `entryId` (UUID, Required): Unique identifier for the knowledge base entry.
- **Example Request**:
  ```bash
  curl -X DELETE https://api.example.com/v1/properties/uuid-of-property/knowledge-base/uuid-of-entry \
  -H "Authorization: Bearer <token>"
  ```
- **Responses**:
  - **204 No Content**
  - **404 Not Found**
    ```json
    {
        "status": "error",
        "code": 404,
        "message": "Knowledge base entry not found."
    }
    ```

### 5. POST /properties/{`propertyId`}/knowledge-base/upload **(inferred)**
- **Description**: Uploads bulk knowledge base entries for a property via a CSV file. This allows hosts to quickly populate or update their property’s knowledge base.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Path Parameters**:
  - `propertyId` (UUID, Required): Unique identifier for the property.
- **Parameters**:
  - `file` (File, Required): CSV file containing knowledge base entries.
- **Example Request**:
  ```bash
  curl -X POST https://api.example.com/v1/properties/uuid-of-property/knowledge-base/upload \
  -H "Authorization: Bearer <token>" \
  -F "file=@path-to-csv-file.csv"
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Bulk upload successful.",
            "uploaded_entries": 10
        }
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Invalid file format or upload failed."
    }
    ```