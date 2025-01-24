### **Endpoint: `/update_property`**

#### **Purpose**
This endpoint allows users to update an existing property, modifying attributes such as the property name or placeholder values. Users can only update properties **associated with their tenant**.

---

## **1. Technical Details**

### **Method**
`PATCH`

### **URL**
`/update_property`

### **Authentication**
Requires a **valid JWT token** to authenticate the user and ensure they have permission to update the specified property.

---

## **2. Request Parameters**

| **Parameter**    | **Type**   | **Required** | **Description** |
|-----------------|-----------|--------------|-----------------|
| `property_id`   | UUID      | Yes          | The unique ID of the property to update. |
| `tenant_id`     | UUID      | Yes (from JWT) | The tenant that owns the property (derived from JWT). |
| `name`          | String    | No           | The updated name of the property. |
| `placeholders`  | Object[]  | No           | A list of updated placeholders (only send changed values). |

### **Example Request:**
```json
{
  "property_id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "Updated Property Name",
  "placeholders": [
    {
      "id": "placeholder-001",
      "value": "Updated Value"
    },
    {
      "id": "placeholder-002",
      "value": "New Phone Number"
    }
  ]
}
```

---

## **3. Endpoint Logic**

### **1. Authentication & Authorization**
- Extract the JWT token.
- Validate the user's **tenant ID** from the token.
- Ensure the property belongs to the user's **tenant**.

### **2. Property Validation**
- Verify that the `property_id` exists in the database.
- Ensure the property belongs to the correct tenant.
- If the `name` is updated, ensure **no duplicate names exist**.

### **3. Update Logic**
- Update the **property name** if provided.
- Update **placeholders** only if included in the request.
- Store an **audit log** entry capturing the change (`last_modified_by` field).

---

## **4. Responses**

### **✅ Success**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `200 OK`       | `"Property updated successfully."` |

**Example Response:**
```json
{
  "status": "success",
  "code": 200,
  "message": "Property updated successfully."
}
```

---

### **❌ Errors**

#### **1️⃣ Property Not Found**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `404 Not Found` | `"Property not found."` |

#### **2️⃣ Unauthorized Access**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `403 Forbidden` | `"Unauthorized: You do not have access to this property."` |

#### **3️⃣ Duplicate Property Name**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `409 Conflict` | `"Property name already exists."` |

#### **4️⃣ Missing Required Fields**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `400 Bad Request` | `"Missing required parameters."` |

---

## **5. Implementation Code**
```python
@v1.route('/update_property', methods=['PATCH'])
@jwt_required()
def update_property():
    """
    Endpoint to update a property's details.
    Requires authentication and tenant validation.
    """

    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /update_property Updating property.")

    try:
        # Extract user and tenant from JWT
        current_user_id = get_jwt_identity()
        tenant_id = get_tenant_from_jwt()

        # Get request data
        data = request.get_json()
        property_id = data.get('property_id')
        new_name = data.get('name')
        updated_placeholders = data.get('placeholders', [])

        # Validate required fields
        if not property_id:
            return utils.jsonify_return_error("error", 400, "Missing property_id."), 400

        # Retrieve property
        property_to_update = Property.query.filter_by(id=property_id, tenant_id=tenant_id).first()

        if not property_to_update:
            return utils.jsonify_return_error("error", 404, "Property not found."), 404

        # Check for duplicate property name
        if new_name and Property.query.filter_by(name=new_name, tenant_id=tenant_id).first():
            return utils.jsonify_return_error("error", 409, "Property name already exists."), 409

        # Apply updates
        if new_name:
            property_to_update.name = new_name

        for placeholder in updated_placeholders:
            placeholder_entry = Placeholder.query.filter_by(id=placeholder["id"], property_id=property_id).first()
            if placeholder_entry:
                placeholder_entry.value = placeholder["value"]

        # Save changes
        db.session.commit()
        current_app.logger.info(f"{user_ip} - /update_property success.")

        return utils.jsonify_return_success("success", 200, {"message": "Property updated successfully."}), 200

    except Exception as e:
        current_app.logger.error(f"{user_ip} - /update_property Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
```

---

## **6. Security Considerations**
- **JWT Required**: Only authenticated users can modify properties.
- **Tenant ID Validation**: Ensures users can **only** update properties within their **own** organization.
- **Duplicate Name Prevention**: Ensures unique property names.
- **Audit Log**: Saves modification history.

---

## **7. Next Steps**
✅ **Integrate frontend UI for property editing.**  
✅ **Test API error handling for unauthorized access.**  
✅ **Optimize query performance for large property sets.**  
