# Endpoint: `/get_properties`

## **1. Purpose**
This endpoint retrieves the list of **properties** owned by a **specific user**.  
It ensures that only **authenticated users** can access properties **within their user_id**.

---

## **2. Technical Specifications**

### **Method**
`GET`

### **URL**
`/get_properties`

### **Authentication**
Requires a **valid JWT token**.  
The response is **scoped to the user's id**.

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
- Decode the token and retrieve the **user ID**.

### **2. Fetch Properties**
- Retrieve **only the properties** associated with the user's id.
- If **no properties exist**, return an empty array.

### **3. Response**
- If properties are found, return them **as a list** (ID, name and description).
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
      "name": "Luxury Villa Rome",
      "description" : "a generic description"
    },
    {
      "property_id": "def456",
      "name": "Downtown Loft",
      "description" : "a generic description"
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

## **6. Updated Implementation Code**
```python
@v1.route('/get_properties', methods=['GET'])
@jwt_required()
def get_properties():
    """
    Retrieves the list of properties for the authenticated user's tenant.
    """

    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /get_properties Getting properties.")

    try:
        current_user_id = get_jwt_identity()
        user = User.query.filter_by(user_id=current_user_id).first()

        if not user:
            current_app.logger.info(f"{user_ip} - /get_properties User not found")
            return utils.jsonify_return_error("error", 404, "User not found"), 404

        properties = Property.query.filter_by(user_id=user.user_id).all()

        if not properties:
            current_app.logger.info(f"{user_ip} - /get_properties No properties found")
            return utils.jsonify_return_success("success", 200, {"data":[]}), 200

        response_data = []
        for property in properties:
            response_data.append({
                "property_id": property.id,
                "name": property.name,
                "description": property.description
            })

        current_app.logger.info(f"{user_ip} - /get_properties Properties retrieved succesfully")
        return utils.jsonify_return_success("success", 200, {"data": response_data}), 200
    
    except ValueError as e:
        current_app.logger.error(f"{user_ip} - /get_properties ValueError. {e}")
        return utils.jsonify_return_error("error", 400, f"/get_properties ValueError {str(e)}"), 400
    except Exception as e:
        current_app.logger.error(f"{user_ip} - /get_properties Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500

```

---

## **7. Security & Business Rules**
✅ **JWT Token Required** → Ensures only authenticated users can retrieve properties.  
<!-- ✅ **Tenant-Based Access Control** → Users only see properties **within their tenant**. -->
✅ **Empty Array for No Properties** → Ensures the API response is always structured properly.  
✅ **Logs & Monitoring** → Tracks property retrieval events.

---

## **8. Next Steps**
🔹 **Apply the `tenant_id` model to the remaining `/property` endpoints**:  
   - `/delete_property`
   - `/update_property`
   - `/duplicate_property`
   - `/get_property_data`

