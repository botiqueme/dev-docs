# **Endpoint: `/delete_placeholder/{property_id}/{placeholder_id}`**

## **Purpose**
This endpoint allows users to delete a placeholder from a specific property. If the placeholder was applied to all properties (`apply_to_all`), users can choose to remove it from **all properties** or only the current one.

---

## **Technical Specifications**

### **Method**
`DELETE`

### **URL**
`/delete_placeholder/{property_id}/{placeholder_id}`

### **Authentication**
🔑 **JWT Token Required**

---

## **Request Parameters**

| **Parameter**       | **Type**  | **Required** | **Description** |
|---------------------|----------|--------------|-----------------|
| `property_id`      | UUID      | ✅ Yes       | The unique identifier of the property where the placeholder exists. |
| `placeholder_id`   | UUID      | ✅ Yes       | The unique identifier of the placeholder to delete. |
| `apply_to_all`     | Boolean   | ❌ Optional  | If `true`, deletes this placeholder from **all properties** where it was applied. Default: `false`. |
| `Authorization`    | Header    | ✅ Yes       | Bearer token to authenticate the user. |

---

## **Example Requests**

### **1️⃣ Remove placeholder from only the given property** (default)
```
DELETE /delete_placeholder/123e4567-e89b-12d3-a456-426614174000/789a4567-e89b-12d3-a456-426614174999
Authorization: Bearer <JWT_TOKEN>
```

### **2️⃣ Remove placeholder from all properties where it was applied**
```
DELETE /delete_placeholder/123e4567-e89b-12d3-a456-426614174000/789a4567-e89b-12d3-a456-426614174999?apply_to_all=true
Authorization: Bearer <JWT_TOKEN>
```

---

## **Implementation Logic**

### **1. Authentication & Authorization**
- Verify JWT token and extract user ID.
- Confirm the user **owns the property**.

### **2. Check if the placeholder exists in the given property**
- If `placeholder_id` does not exist → return **404 Not Found**.

### **3. Handle `apply_to_all` logic**
- If `apply_to_all=false` → delete the placeholder **only from the given property**.
- If `apply_to_all=true` → find **all properties** where the placeholder exists and remove it.

### **4. Delete the placeholder**
- If **only one property was affected**, return `"Placeholder deleted successfully."`
- If **multiple properties were affected**, return `"Placeholder deleted from all properties."`

### **5. Audit Logging**
- Log deletion attempts for debugging & security.

---

## **Response Codes**

| **Scenario** | **HTTP Status** | **Message** |
|-------------|----------------|-------------|
| ✅ **Success (Single Property)** | `200 OK` | `"Placeholder deleted successfully."` |
| ✅ **Success (All Properties)** | `200 OK` | `"Placeholder deleted from all properties."` |
| ❌ **Property Not Found** | `404 Not Found` | `"Property not found."` |
| ❌ **Placeholder Not Found** | `404 Not Found` | `"Placeholder not found."` |
| ❌ **Unauthorized** | `403 Forbidden` | `"You do not have permission to modify this property."` |
| ❌ **Invalid or Missing JWT** | `401 Unauthorized` | `"Authorization token required."` |
| ❌ **Rate Limit Exceeded** | `429 Too Many Requests` | `"Too many deletion requests. Try again later."` |

---

## **Implementation Code**
```python
@v1.route('/delete_placeholder/<string:property_id>/<string:placeholder_id>', methods=['DELETE'])
@jwt_required()
@limiter.limit("10 per hour")
def delete_placeholder(property_id, placeholder_id):
    """
    Deletes a placeholder from a property. If apply_to_all is set to True,
    removes the placeholder from all properties where it was applied.
    """

    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /delete_placeholder Deleting placeholder.")

    try:
        # Extract user ID from JWT token
        current_user_id = get_jwt_identity()

        # Check if the property exists and belongs to the user
        property = Property.query.filter_by(id=property_id, user_id=current_user_id).first()
        if not property:
            current_app.logger.info(f"{user_ip} - /delete_placeholder Property not found")
            return utils.jsonify_return_error("error", 404, "Property not found"), 404

        # Check if the placeholder exists within the property
        placeholder = Placeholder.query.filter_by(id=placeholder_id, property_id=property_id).first()
        if not placeholder:
            current_app.logger.info(f"{user_ip} - /delete_placeholder Placeholder not found")
            return utils.jsonify_return_error("error", 404, "Placeholder not found"), 404

        # Check if "apply_to_all" flag is set in query params
        apply_to_all = request.args.get('apply_to_all', 'false').lower() == 'true'

        if apply_to_all:
            # Get all properties where the placeholder exists
            affected_properties = Placeholder.query.filter_by(name=placeholder.name, user_id=current_user_id).all()

            if affected_properties:
                # Remove the placeholder from all properties
                for p in affected_properties:
                    db.session.delete(p)
                db.session.commit()

                current_app.logger.info(f"{user_ip} - /delete_placeholder Placeholder removed from all properties.")
                return utils.jsonify_return_success("success", 200, {"message": "Placeholder deleted from all properties."}), 200
            else:
                current_app.logger.info(f"{user_ip} - /delete_placeholder No properties found for global removal.")
                return utils.jsonify_return_error("error", 404, "Placeholder not found in any property."), 404

        # If apply_to_all=False, delete only from this property
        db.session.delete(placeholder)
        db.session.commit()

        current_app.logger.info(f"{user_ip} - /delete_placeholder Placeholder removed from single property.")
        return utils.jsonify_return_success("success", 200, {"message": "Placeholder deleted successfully."}), 200

    except Exception as e:
        current_app.logger.error(f"{user_ip} - /delete_placeholder Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
```

---

## **Security & Best Practices**
- ✅ **JWT Authentication**: Ensures only authorized users can delete placeholders.
- ✅ **Ownership Validation**: Users can only modify properties they own.
- ✅ **Rate Limiting**: Maximum **10 deletions per hour** to prevent abuse.
- ✅ **Audit Logging**: Tracks deletion requests for security purposes.

---

## **Key Features**
- **Supports individual and bulk deletion** (`apply_to_all`).
- **Ensures only the owner can delete placeholders**.
- **Includes detailed response messages** for developers and users.

---

### **Next Steps**
- **Frontend Handling**: UI should prompt users to confirm whether they want to delete the placeholder **from a single property** or **from all properties**.
- **Admin Overrides (Optional)**: Consider allowing **admins** to delete placeholders across **all properties** in a system-wide manner.

---

✅ **This version is finalized. Ready to move to the next endpoint.** 🚀
