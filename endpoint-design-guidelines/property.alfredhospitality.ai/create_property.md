# **Endpoint: `/create_property`**

## **1. Purpose**
This endpoint allows users to **create a new property**. The property will be **linked to the `user_id`**

<! Future implementation: The property will be **linked to the `tenant_id`** of the authenticated user, ensuring that all users within the same tenant can access and manage it.>

---

## **2. Technical Specifications**

### **Method**
`POST`

### **URL**
`/create_property`

### **Authentication**
Requires a valid **JWT token**.  
The property will be associated with the **user_id** of the authenticated user.

---

## **3. Request Parameters**

| **Parameter**  | **Type**  | **Required** | **Description** |
|---------------|-----------|--------------|-----------------|
| `Authorization` | Header (JWT) | Yes | Bearer token to authenticate the user. |
| `name` | String | Yes | Unique property name (no duplicates allowed). |

### **Example Request**
```json
POST /create_property
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "name": "Luxury Villa Rome"
}
```

---

## **4. Endpoint Logic**

### **1. JWT Authentication**
- Extract the **JWT token** from the `Authorization` header.
- Decode the token and retrieve the **user ID** <! Future implementation: and **tenant ID**>.

### **2. Property Name Validation**
- Ensure the **`name`** is provided.
- Check that the property **name does not already exist** within the same `tenant_id`.

### **3. Property Creation**
- Assign the new property to the **tenant** of the user.
- Save the property in the database.

### **4. Response**
- If successful, return `201 Created` with the **property ID**.
- If the property name **already exists**, return `409 Conflict`.

---

## **5. Responses**

### ✅ **Success**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `201 Created`  | `"Property created successfully."` |

**Response Example:**
```json
{
  "status": "success",
  "code": 201,
  "data": {
    "property_id": "abc123",
    "name": "Luxury Villa Rome",
    "tenant_id": "tenant_456"
  }
}
```

---

### ❌ **Errors**

#### **1️⃣ Missing Property Name**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `400 Bad Request` | `"Missing property name."` |

**Response Example:**
```json
{
  "status": "error",
  "code": 400,
  "message": "Missing property name."
}
```

---

#### **2️⃣ Property Name Already Exists (within the same tenant)**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `409 Conflict` | `"A property with this name already exists in your tenant."` |

**Response Example:**
```json
{
  "status": "error",
  "code": 409,
  "message": "A property with this name already exists in your tenant."
}
```

---

#### **3️⃣ Unauthorized User (No JWT Provided)**
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
@v1.route('/create_property', methods=['POST'])
@jwt_required()
def create_property():
    """
    Creates a new property for the authenticated user's tenant.
    """

    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /create_property Creating new property.")

    try:
        # Extract user and tenant information from the JWT token
        current_user_id = get_jwt_identity()
        user = User.query.filter_by(user_id=current_user_id).first()

        if not user:
            return jsonify_return_error("error", 404, "User not found"), 404

        tenant_id = user.tenant_id  # Ensure we use the tenant ID

        # Parse request data
        data = request.get_json()
        property_name = data.get("name")

        # Validate property name
        if not property_name:
            return jsonify_return_error("error", 400, "Missing property name."), 400

        # Ensure property name is unique within the tenant
        existing_property = Property.query.filter_by(name=property_name, tenant_id=tenant_id).first()
        if existing_property:
            return jsonify_return_error("error", 409, "A property with this name already exists in your tenant."), 409

        # Create the property
        new_property = Property(
            tenant_id=tenant_id,
            name=property_name
        )

        db.session.add(new_property)
        db.session.commit()

        # Prepare success response
        response_data = {
            "property_id": new_property.property_id,
            "name": new_property.name,
            "tenant_id": new_property.tenant_id
        }

        current_app.logger.info(f"{user_ip} - /create_property success Property created")
        return jsonify_return_success("success", 201, response_data), 201

    except Exception as e:
        current_app.logger.error(f"{user_ip} - /create_property Internal Server Error. {e}")
        return jsonify_return_error("error", 500, "Internal Server Error."), 500
```

---

## **7. Security & Business Rules**
✅ **JWT Token Required** → Ensures only authenticated users can create properties.  
✅ **Unique Property Name Per Tenant** → Prevents duplicate names within the same tenant.  
✅ **Logs & Monitoring** → Tracks property creation events.  
✅ **No Property Limits (for now)** → Users can create unlimited properties.

---

## **8. Next Steps**
🔹 **Apply the `tenant_id` model to the remaining `/property` endpoints**:  
   - `/get_properties`
   - `/delete_property`
   - `/update_property`
   - `/duplicate_property`
   - `/get_property_data`

🔹 **Consider introducing role-based access control (RBAC) in the future**:  
   - Admin vs. Standard Users in a tenant.

