# **Endpoint: `/update_placeholder/{property_id}/{placeholder_id}`**

## **Purpose**
This endpoint allows updating the value of an **existing placeholder** in a specified property.

---

## **Technical Specifications**

### **Method**
`PATCH`

### **URL**
`/update_placeholder/{property_id}/{placeholder_id}`

### **Authentication**
🔑 **JWT Token Required (Access Token)**

---

## **Request Parameters**

| **Parameter**       | **Type**  | **Required** | **Description** |
|---------------------|-----------|--------------|-----------------|
| `property_id`      | Path      | ✅ Yes        | The unique ID of the property where the placeholder exists. |
| `placeholder_id`   | Path      | ✅ Yes        | The unique ID of the placeholder to be updated. |
| `value`            | JSON Body | ✅ Yes        | The new value for the placeholder (type varies based on placeholder type). |

### **Example Request**
```
PATCH /update_placeholder/123e4567-e89b-12d3-a456-426614174000/9876543210abcdef
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "value": "New Placeholder Value"
}
```

---

## **Validation & Logic**

1. **Authentication & Authorization**
   - Validate the JWT token.
   - Ensure the **user has permission** to modify the property.

2. **Verify Property and Placeholder**
   - Check if the `property_id` exists and belongs to the authenticated user.
   - Check if the `placeholder_id` exists within the given property.

3. **Validate Input**
   - Ensure the `value` provided matches the expected type for the placeholder.
   - Enforce **any constraints or formatting** based on placeholder type (e.g., phone number regex validation, max length for text).

4. **Update the Placeholder**
   - Modify the existing placeholder with the new value.
   - Log changes for auditing.

5. **Respond with Confirmation**
   - Return a success response with the updated placeholder details.

---

## **Response Codes**

| **Scenario**          | **HTTP Status** | **Message** |
|----------------------|----------------|-------------|
| ✅ **Success** | `200 OK` | `"Placeholder updated successfully."` |
| ❌ **Property Not Found** | `404 Not Found` | `"Property not found."` |
| ❌ **Placeholder Not Found** | `404 Not Found` | `"Placeholder not found in this property."` |
| ❌ **Invalid Input** | `400 Bad Request` | `"Invalid value format for this placeholder."` |
| ❌ **Unauthorized** | `403 Forbidden` | `"You do not have permission to update this placeholder."` |
| ❌ **Rate Limit Exceeded** | `429 Too Many Requests` | `"Too many requests. Please try again later."` |

---

## **Implementation Code**
```python
@v1.route('/update_placeholder/<property_id>/<placeholder_id>', methods=['PATCH'])
@jwt_required()
def update_placeholder(property_id, placeholder_id):
    """
    Updates the value of an existing placeholder in a property.
    Requires JWT authentication.
    """
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /update_placeholder/{property_id}/{placeholder_id} - Updating placeholder.")

    try:
        # Get user ID from JWT
        current_user_id = get_jwt_identity()

        # Retrieve property
        property_instance = Property.query.filter_by(id=property_id, user_id=current_user_id).first()
        if not property_instance:
            current_app.logger.info(f"{user_ip} - /update_placeholder Property not found.")
            return utils.jsonify_return_error("error", 404, "Property not found."), 404

        # Retrieve placeholder
        placeholder = Placeholder.query.filter_by(id=placeholder_id, property_id=property_id).first()
        if not placeholder:
            current_app.logger.info(f"{user_ip} - /update_placeholder Placeholder not found.")
            return utils.jsonify_return_error("error", 404, "Placeholder not found in this property."), 404

        # Validate input
        data = request.get_json()
        new_value = data.get("value")
        if not new_value:
            return utils.jsonify_return_error("error", 400, "Missing value."), 400

        # Validate placeholder type and constraints
        if not utils.validate_placeholder_value(placeholder, new_value):
            return utils.jsonify_return_error("error", 400, "Invalid value format for this placeholder."), 400

        # Update the placeholder
        placeholder.value = new_value
        db.session.commit()

        current_app.logger.info(f"{user_ip} - /update_placeholder/{property_id}/{placeholder_id} - Placeholder updated successfully.")
        return utils.jsonify_return_success("success", 200, {"message": "Placeholder updated successfully."}), 200

    except Exception as e:
        current_app.logger.error(f"{user_ip} - /update_placeholder Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
```

---

## **Security & Best Practices**
✅ **Authentication Required**: Only authenticated users can modify placeholders.  
✅ **Ownership Validation**: Users can only update placeholders within their own properties.  
✅ **Strict Input Validation**: Ensures valid data formats based on placeholder type.  
✅ **Logging & Monitoring**: All updates are logged for security and auditing.  

---

## **Optional Improvements**
🔍 **Atomic Transactions & Race Conditions Prevention**  
If multiple updates to the same placeholder occur simultaneously, data inconsistency could happen.  
**Solution:** Use **atomic transactions** or implement a **versioning mechanism** to prevent race conditions.

---

## **Next Steps**
- **Frontend Updates:** Ensure UI provides validation feedback.  
- **Extended Error Handling:** Add more granular error messages if needed.  
- **Performance Monitoring:** Track update request frequency for optimization.  

---

