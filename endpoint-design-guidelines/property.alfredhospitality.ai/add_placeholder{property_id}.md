### **Endpoint: `/add_placeholder/{property_id}`**

#### **Purpose**
Allows a user to add a new placeholder to a specific property. Placeholders can store text, images, videos, phone numbers, or other custom data for the property.

---

## **1. Technical Specifications**

### **Method**
`POST`

### **URL**
`/add_placeholder/{property_id}`

### **Authentication**
🔑 **JWT (Access Token Required)**  
The request must be authenticated, and the user must have access to the specified property.

---

## **2. Request Parameters**

### **Path Parameter**
| **Parameter**  | **Type**  | **Required** | **Description**                       |
|---------------|-----------|--------------|---------------------------------------|
| `property_id` | Path      | Yes          | The unique identifier of the property.|

### **Request Body**
| **Field**      | **Type** | **Required** | **Description**                       |
|----------------|----------|--------------|---------------------------------------|
| `name`         | String   | Yes          | The name of the placeholder.          |
| `type`         | String   | Yes          | The type of the placeholder. Allowed values: `text`, `number`, `email`, `address`, `image`, `video`, or `custom`. |
| `value`        | String   | Yes          | The value of the placeholder.         |
| `apply_to_all` | Boolean  | No           | If true, the placeholder is applied to all properties owned by the user. |

**Example Request:**
```json
{
  "name": "Welcome Message",
  "type": "text",
  "value": "Welcome to our property!",
  "apply_to_all": false
}
```

---

## **3. Response**

### ✅ **Success**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `201 Created`   | "Placeholder added successfully." |

#### **Example Response:**
```json
{
  "status": "success",
  "code": 201,
  "data": {
    "message": "Placeholder added successfully.",
    "placeholder_id": "abc123",
    "property_id": "123e4567-e89b-12d3-a456-426614174000"
  }
}
```

---

### ❌ **Errors**

#### **1️⃣ Property Not Found**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `404 Not Found` | "Property not found." |

**Example Response:**
```json
{
  "status": "error",
  "code": 404,
  "message": "Property not found."
}
```

---

#### **2️⃣ Placeholder Name Already Exists**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `409 Conflict`  | "Placeholder name already exists." |

**Example Response:**
```json
{
  "status": "error",
  "code": 409,
  "message": "Placeholder name already exists."
}
```

---

#### **3️⃣ Unauthorized Access**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `403 Forbidden` | "Access denied. You do not have permission to modify this property." |

**Example Response:**
```json
{
  "status": "error",
  "code": 403,
  "message": "Access denied. You do not have permission to modify this property."
}
```

---

#### **4️⃣ Invalid Token**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `401 Unauthorized` | "Invalid or expired token." |

**Example Response:**
```json
{
  "status": "error",
  "code": 401,
  "message": "Invalid or expired token."
}
```

---

## **4. Implementation Code**

```python
@v1.route('/add_placeholder/<property_id>', methods=['POST'])
@jwt_required()
def add_placeholder(property_id):
    """
    Adds a new placeholder to a specific property.
    Requires JWT authentication.
    """

    user_id = get_jwt_identity()

    # Fetch property
    property_instance = Property.query.filter_by(id=property_id, user_id=user_id).first()

    if not property_instance:
        return jsonify_return_error("error", 404, "Property not found."), 404

    # Parse request data
    data = request.get_json()
    name = data.get("name")
    placeholder_type = data.get("type")
    value = data.get("value")
    apply_to_all = data.get("apply_to_all", False)

    # Validate inputs
    if not name or not placeholder_type or not value:
        return jsonify_return_error("error", 400, "Missing required fields: name, type, or value."), 400

    # Check for duplicate placeholder name in the same property
    existing_placeholder = Placeholder.query.filter_by(property_id=property_id, name=name).first()
    if existing_placeholder:
        return jsonify_return_error("error", 409, "Placeholder name already exists."), 409

    # Add placeholder to the property
    new_placeholder = Placeholder(
        property_id=property_id,
        name=name,
        type=placeholder_type,
        value=value
    )
    db.session.add(new_placeholder)

    # Apply placeholder to all properties (if applicable)
    if apply_to_all:
        user_properties = Property.query.filter_by(user_id=user_id).all()
        for user_property in user_properties:
            if user_property.id != property_id:
                placeholder_copy = Placeholder(
                    property_id=user_property.id,
                    name=name,
                    type=placeholder_type,
                    value=value
                )
                db.session.add(placeholder_copy)

    db.session.commit()

    # Return response
    return jsonify_return_success("success", 201, {
        "message": "Placeholder added successfully.",
        "placeholder_id": new_placeholder.id,
        "property_id": property_id
    }), 201
```

---

## **5. Security Considerations**
1. **JWT Authentication**
   - Ensures only authorized users can add placeholders to their properties.
2. **Input Validation**
   - Verifies that required fields are present and correctly formatted.
3. **Duplicate Placeholder Prevention**
   - Ensures placeholder names are unique within a property.

---

## **6. Next Steps**
- Validate placeholder types (`text`, `number`, etc.) to match the allowed formats.
- Implement frontend messages to guide users in the creation process.

---

## 🔍 **Optional Improvement (to evaluate):**  
### **Potential Race Condition on `apply_to_all`**  
If multiple concurrent requests attempt to add the same placeholder to multiple properties, we might unintentionally create duplicates.

💡 **Solution:**  
1. **Use Atomic Transactions:**  
   - Atomic transactions ensure that **all operations either succeed together or fail entirely**.  
   - This prevents a scenario where some placeholders are added while others fail, ensuring **data consistency**.  
   - If an error occurs during the process, the entire transaction **rolls back**, leaving the database unchanged.

2. **Implement a Unique Check at Commit Time:**  
   - Before committing the changes, perform a **uniqueness validation** to check if the placeholder already exists in the target properties.  
   - This prevents duplicate entries and **ensures idempotency** (same request produces the same result even if repeated).  

**Choosing the Right Approach:**  
- **Atomic transactions** are recommended when working with databases that support them (e.g., PostgreSQL, MySQL with InnoDB).  
- **Commit-time uniqueness checks** are useful if the database or ORM does not support full transaction isolation.
