# **Endpoint: `/get_property_data`**  

#### **Purpose**  
This endpoint retrieves detailed information about a **specific property** belonging to a **specific tenant**, including all associated **placeholders** and their values.  

---

### **Technical Details**  

- **Method:** `GET`  
- **URL:** `/get_property_data`  
- **Authentication:** Requires **JWT Token**  
- **Scope:** Only returns properties that belong to the **authenticated user’s tenant**  

---

### **Request Parameters**  

| **Parameter**  | **Type**  | **Required** | **Description** |
|---------------|-----------|--------------|-----------------|
| `property_id` | UUID      | ✅ Yes       | The unique identifier of the property to retrieve. |
| `tenant_id`   | UUID      | ❌ No       | Derived from JWT; ensures only properties within the user's tenant are retrieved. |

#### **Example Request**  
```
GET /get_property_data?property_id=123e4567-e89b-12d3-a456-426614174000
Authorization: Bearer <JWT_TOKEN>
```

---

### **Endpoint Logic**  

1. **JWT Authentication**  
   - Extract the JWT token from the `Authorization` header.  
   - Decode the JWT and retrieve the `tenant_id` linked to the authenticated user.  

2. **Property Validation**  
   - Ensure the provided `property_id` belongs to the **same tenant** as the user.  
   - If the property does **not exist** or does **not belong to the tenant**, return a `404 Not Found`.  

3. **Retrieve Property Data**  
   - Fetch the **property details**, including:
     - Property **name**
     - Property **status** (`draft`, `active`, `inactive`, `archived`)
     - List of **placeholders** and their values (standard + custom)
     - **Created by** and **Last modified by**
     - **Subscription status** (if applicable)

4. **Response Handling**  
   - If the request is valid, return the **full property details**.  
   - If the property is **archived**, return a warning message that it may be deleted soon.  

---

### **Response Codes & Examples**  

#### ✅ **Success** (`200 OK`)  
```json
{
  "status": "success",
  "code": 200,
  "data": {
    "property_id": "123e4567-e89b-12d3-a456-426614174000",
    "tenant_id": "9f2345ff-3b67-41c8-9223-1a2b3c4d5e6f",
    "name": "Luxury Apartment Rome",
    "status": "active",
    "placeholders": [
      {
        "name": "wifi_password",
        "type": "text",
        "value": "MySecureWiFi123"
      },
      {
        "name": "owner_phone",
        "type": "phone",
        "value": "+39 333 1234567"
      }
    ],
    "created_by": "John Doe",
    "last_modified_by": "Jane Doe",
    "subscription_active": true
  }
}
```

---

#### ❌ **Errors**  

| **Scenario**                 | **HTTP Status** | **Message**  |
|------------------------------|----------------|--------------|
| **Property Not Found**        | `404 Not Found` | `"Property not found or does not belong to your account."` |
| **Unauthorized Access**       | `401 Unauthorized` | `"Authorization token required."` |
| **Account Inactive**          | `403 Forbidden` | `"Your account is inactive. Please reactivate to access property data."` |
| **Subscription Expired**      | `403 Forbidden` | `"Subscription expired. Reactivate to retrieve property data."` |

---

### **Implementation Code**
```python
@v1.route('/get_property_data', methods=['GET'])
@jwt_required()
def get_property_data():
    """
    Retrieves detailed information about a specific property.
    Requires authentication via JWT.
    """
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /get_property_data Fetching property details.")

    try:
        # Extract user and tenant information from JWT
        current_user_id = get_jwt_identity()
        current_user = User.query.filter_by(user_id=current_user_id).first()

        if current_user is None:
            current_app.logger.info(f"{user_ip} - /get_property_data User not found")
            return utils.jsonify_return_error("error", 404, "User not found"), 404

        if not current_user.is_active:
            return utils.jsonify_return_error("error", 403, "Your account is inactive. Please reactivate to access property data."), 403

        # Extract property ID from request
        property_id = request.args.get("property_id")

        if not property_id:
            return utils.jsonify_return_error("error", 400, "Missing property_id parameter."), 400

        # Retrieve property and validate ownership
        property_instance = Property.query.filter_by(property_id=property_id, tenant_id=current_user.tenant_id).first()

        if not property_instance:
            return utils.jsonify_return_error("error", 404, "Property not found or does not belong to your account."), 404

        # Prepare response data
        response_data = {
            "property_id": property_instance.property_id,
            "tenant_id": property_instance.tenant_id,
            "name": property_instance.name,
            "status": property_instance.status,
            "placeholders": [
                {"name": p.name, "type": p.type, "value": p.value} for p in property_instance.placeholders
            ],
            "created_by": property_instance.created_by,
            "last_modified_by": property_instance.last_modified_by,
            "subscription_active": property_instance.subscription_active
        }

        current_app.logger.info(f"{user_ip} - /get_property_data success fetching property details")
        return utils.jsonify_return_success("success", 200, response_data), 200

    except Exception as e:
        current_app.logger.error(f"{user_ip} - /get_property_data Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
```

---

### **Security & Best Practices**  
✅ **JWT Authentication** → Ensures only authorized users can access property data.  
✅ **Ownership Validation** → Prevents users from accessing data of other tenants.  
✅ **Rate Limiting (Optional)** → Avoids abuse of API calls.  
✅ **Logging & Monitoring** → Tracks property access requests for security audits.  

---

### **Next Steps**  
1️⃣ **Confirm if subscription data should be included.**  
2️⃣ **Implement UI/Frontend handling for inactive users.**  
3️⃣ **Ensure placeholders support all expected data types (text, media, etc.).**  
