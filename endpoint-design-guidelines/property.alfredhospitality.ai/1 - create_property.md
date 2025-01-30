# **Endpoint: `/create_property`**

## **1. Purpose**
This endpoint allows users to **create a new property**. The property will be **linked to the `user_id`**

<!-- Future implementation: The property will be **linked to the `tenant_id`** of the authenticated user, ensuring that all users within the same tenant can access and manage it.-->
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
- Decode the token and retrieve the **user ID** <!-- Future implementation: and **tenant ID**-->.

### **2. Property Name Validation**
- Ensure the **`name`** is provided.
- Check that the property **name does not already exist** within the same `user_id`.

### **3. Property Creation**
- Assign the new property to the **user_id** of the user.
- Save the property in the database `property`.

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
    "name": "Luxury Villa Rome"
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
    '''Create a new property for the user'''

    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /create_property Creating new property.")

    try:
        current_user_id = get_jwt_identity()
        user = User.query.filter_by(user_id=current_user_id).first()

        if not user:
            current_app.logger.info(f"{user_ip} - /create_property User not found")
            return utils.jsonify_return_error("error", 404, "User not found"), 404

        data = request.get_json()

        property_name = data.get("name")
        if not property_name:
            current_app.logger.info(f"{user_ip} - /create_property Missing property name")
            return utils.jsonify_return_error("error", 400, "Missing property name"), 400
        

        # Check if the user has already a property with the same name
        property = Property.query.filter_by(user_id=user.user_id, name=property_name).first()

        if property:
            current_app.logger.info(f"{user_ip} - /create_property A property with this name already exists")
            return utils.jsonify_return_error("error", 409, "A property with this name already exists"), 409
        
        # Create the property
        new_property = Property(
            user_id=user.user_id,
            name=property_name,
            description=data.get("description")
        )

        db.session.add(new_property)
        db.session.commit()

        response_data = {
            "property_id": property.id,
            "name": property.name,
        }

        current_app.logger.info(f"{user_ip} - /create_property success Property created")
        return utils.jsonify_return_success("success", 201, response_data), 201
    
    except ValueError as e:
        return utils.jsonify_return_error("error", 400, f"/create_property ValueError {str(e)}")
    except Exception as e:
        current_app.logger.error(f"{user_ip} - /create_property Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500

```

---

## **7. Security & Business Rules**
✅ **JWT Token Required** → Ensures only authenticated users can create properties.  
✅ **Unique Property Name Per User** → Prevents duplicate names within the same user.  
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

