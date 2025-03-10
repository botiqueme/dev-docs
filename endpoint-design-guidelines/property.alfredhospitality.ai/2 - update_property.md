### **Endpoint: `/update_property`**

#### **Purpose**
This endpoint allows users to update an existing property, modifying attributes such as the property name or description values. Users can only update properties **associated with their user_id**.

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
| `property_name`          | String    | No           | The updated name of the property. |
| `description`   | String    | No           | The updated description for the property |

### **Example Request:**
```json
{
  "property_id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "Updated Property Name",
  "description": "Updated description value" 
}
```

---

## **3. Endpoint Logic**

### **1. Authentication & Authorization**
- Extract the JWT token.
- Validate the user's **user ID** from the token.
- Ensure the property belongs to the user's **id**.

### **2. Property Validation**
- Verify that the `property_id` exists in the database.
- Ensure the property belongs to the correct tenant.
- If the `name` is updated, ensure **no duplicate names exist**.

### **3. Update Logic**
- Update the **property name** if provided.
- Update **description** only if included in the request.
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
        current_user_id = get_jwt_identity()
        user = User.query.filter_by(user_id=current_user_id).first()

        if not user:
            current_app.logger.info(f"{user_ip} - /update_property User not found")
            return utils.jsonify_return_error("error", 404, "User not found"), 404

        data = request.get_json()
        property_id = data.get("property_id")
        new_name = data.get("name")
        new_description = data.get("description")

        #validate required fields
        if not property_id:
            current_app.logger.info(f"{user_ip} - /update_property Missing property_id")
            return utils.jsonify_return_error("error", 400, "Missing property_id"), 400
        
        property_to_update = Property.query.filter_by(user_id=user.user_id, property_id=property_id).first()

        if not property_to_update:
            current_app.logger.info(f"{user_ip} - /update_property Property not found")
            return utils.jsonify_return_error("error", 404, "Property not found"), 404
        
        #check for duplicate properties name
        if new_name:
            property_with_same_name = Property.query.filter_by(user_id=user.user_id, name=new_name).first()
            if property_with_same_name and property_with_same_name.id != property_id:
                current_app.logger.info(f"{user_ip} - /update_property A property with this name already exists")
                return utils.jsonify_return_error("error", 409, "A property with this name already exists"), 409
        
        #update property details
        if new_name:
            property_to_update.name = new_name
        if new_description:
            property_to_update.description = new_description

        db.session.commit()
        
        current_app.logger.info(f"{user_ip} - /update_property success Property updated")
        return utils.jsonify_return_success("success", 200, {"message": "Property updated successfully."}), 200
    
    except ValueError as e:
        current_app.logger.error(f"{user_ip} - /update_property ValueError. {e}")
        return utils.jsonify_return_error("error", 400, f"/update_property ValueError {str(e)}"), 400
    except Exception as e:
        current_app.logger.error(f"{user_ip} - /update_property Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500

```

---

## **6. Security Considerations**
- **JWT Required**: Only authenticated users can modify properties.
- **user ID Validation**: Ensures users can **only** update properties within their **own** organization.
- **Duplicate Name Prevention**: Ensures unique property names.
- **Audit Log**: Saves modification history.

---

## **7. Next Steps**
✅ **Integrate frontend UI for property editing.**  
✅ **Test API error handling for unauthorized access.**  
✅ **Optimize query performance for large property sets.**  
