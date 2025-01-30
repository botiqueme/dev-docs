# Endpoint: `/get_properties`**

## **1. Purpose**
This endpoint retrieves the list of **properties** owned by a **specific tenant**.  
It ensures that only **authenticated users** can access properties **within their tenant**.

---

## **2. Technical Specifications**

### **Method**
`GET`

### **URL**
`/get_properties`

### **Authentication**
Requires a **valid JWT token**.  
The response is **scoped to the user's `tenant_id`**.

---

## **3. Request Parameters**

| **Parameter**  | **Type**  | **Required** | **Description** |
|---------------|-----------|--------------|-----------------|
| `Authorization` | Header (JWT) | Yes | Bearer token to authenticate the user. |

### **Example Request**
```
GET /get_properties
Authorization: Bearer <JWT_TOKEN>
```

---

## **4. Endpoint Logic**

### **1. JWT Authentication**
- Extract the **JWT token** from the `Authorization` header.
- Decode the token and retrieve the **user ID** and **tenant ID**.

### **2. Fetch Properties**
- Retrieve **only the properties** associated with the user's **`tenant_id`**.
- If **no properties exist**, return an empty array.

### **3. Response**
- If properties are found, return them **as a list** (ID & name only).
- If the user **does not belong to a tenant**, return `403 Forbidden`.

---

## **5. Responses**

### ✅ **Success**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `200 OK`  | `"Properties retrieved successfully."` |

**Response Example (Properties Found):**
```json
{
  "status": "success",
  "code": 200,
  "data": [
    {
      "property_id": "abc123",
      "name": "Luxury Villa Rome"
    },
    {
      "property_id": "def456",
      "name": "Downtown Loft"
    }
  ]
}
```

---

### ❌ **Errors**

#### **1️⃣ No Properties Found**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `200 OK` | `"No properties found."` |

**Response Example:**
```json
{
  "status": "success",
  "code": 200,
  "data": []
}
```

---

#### **2️⃣ Unauthorized User (No JWT Provided)**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `401 Unauthorized` | `"Authorization token required."` |

**Response Example:**
```json
{
  "status": "error",
  "code": 401,
  "message": "Authorization token required."
}
```

---

#### **3️⃣ User Not Assigned to a Tenant**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `403 Forbidden` | `"User does not belong to a tenant."` |

**Response Example:**
```json
{
  "status": "error",
  "code": 403,
  "message": "User does not belong to a tenant."
}
```

---

## **6. Updated Implementation Code**
```python
@v1.route('/get_properties', methods=['GET'])
@jwt_required()
def get_properties():
    """
    Retrieves the list of properties for the authenticated user's tenant.
    """

    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /get_properties Retrieving properties.")

    try:
        # Extract user and tenant information from JWT token
        current_user_id = get_jwt_identity()
        user = User.query.filter_by(user_id=current_user_id).first()

        if not user:
            return jsonify_return_error("error", 404, "User not found"), 404

        tenant_id = user.tenant_id  # Ensure properties are scoped to the tenant

        if not tenant_id:
            return jsonify_return_error("error", 403, "User does not belong to a tenant."), 403

        # Retrieve properties for the tenant
        properties = Property.query.filter_by(tenant_id=tenant_id).all()

        if not properties:
            return jsonify_return_success("success", 200, {"data": []}), 200

        # Format response
        properties_data = [
            {
                "property_id": prop.property_id,
                "name": prop.name
            }
            for prop in properties
        ]

        current_app.logger.info(f"{user_ip} - /get_properties success Properties retrieved.")
        return jsonify_return_success("success", 200, {"data": properties_data}), 200

    except Exception as e:
        current_app.logger.error(f"{user_ip} - /get_properties Internal Server Error. {e}")
        return jsonify_return_error("error", 500, "Internal Server Error."), 500
```

---

## **7. Security & Business Rules**
✅ **JWT Token Required** → Ensures only authenticated users can retrieve properties.  
✅ **Tenant-Based Access Control** → Users only see properties **within their tenant**.  
✅ **Empty Array for No Properties** → Ensures the API response is always structured properly.  
✅ **Logs & Monitoring** → Tracks property retrieval events.

---

## **8. Next Steps**
🔹 **Apply the `tenant_id` model to the remaining `/property` endpoints**:  
   - `/delete_property`
   - `/update_property`
   - `/duplicate_property`
   - `/get_property_data`

