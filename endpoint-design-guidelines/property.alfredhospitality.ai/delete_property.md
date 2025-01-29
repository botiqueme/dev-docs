# **Endpoint: `/delete_property`**

## **Purpose**
This endpoint allows users to **permanently delete** a property from their account.  
⚠ **Hard delete only** – once deleted, the property **cannot be recovered**.

## **Technical Details**

### **Method**
`DELETE`

### **URL**
`/delete_property`

### **Authentication**
✅ **JWT Required** (User must be authenticated).  
✅ **Tenant ID Retrieved from JWT** (to ensure the property belongs to the correct project).  

---

## **Request Parameters**

| **Parameter**  | **Type**  | **Required** | **Description** |
|---------------|-----------|--------------|-----------------|
| `property_id` | String    | ✅ Yes      | The UUID of the property to be deleted. |
| `confirm_name` | String   | ✅ Yes       | The **exact** name of the property, required to confirm deletion (GitHub-style protection). |

🔹 **Security Feature**: The `confirm_name` field prevents accidental deletions by requiring the user to manually type the property name instead of copy-pasting.

**Example Request:**
```
DELETE /delete_property
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "property_id": "456e7890-fghj-12d3-a456-426614174999",
  "confirm_name": "My Rental Property"
}
```

---

## **Response Codes**

| **Scenario**              | **HTTP Status** | **Message** |
|--------------------------|----------------|-------------|
| **Success**              | `200 OK`       | `"Property deleted successfully."` |
| **Property Not Found**   | `404 Not Found` | `"Property not found."` |
| **Incorrect Confirmation Name** | `400 Bad Request` | `"Property name does not match. Please enter the correct name to delete."` |
| **Unauthorized Access**  | `403 Forbidden` | `"You are not authorized to delete this property."` |
| **Rate Limit Exceeded**  | `429 Too Many Requests` | `"Too many deletion attempts. Please try again later."` |

---

## **Implementation Logic**

### **1. JWT Authentication & Tenant Validation**
- Extract the **tenant_id** from the **JWT**.
- Retrieve the property from the database using **property_id** and **tenant_id**.
- If the property doesn’t belong to the user’s project, return **403 Forbidden**.

### **2. Confirm Property Name**
- Compare the provided `confirm_name` with the actual name of the property in the database.
- If they don’t match, return **400 Bad Request**.

### **3. Perform Hard Delete**
- If all checks pass, **delete the property permanently**.

### **4. Send Confirmation Email (optional)**
- Notify the user via email that the property has been deleted.

### **5. Logging & Rate Limiting**
- Log every deletion attempt.
- Rate limit to **3 attempts per hour per user** to prevent abuse.

---

## **Code Implementation**
```python
@v1.route('/delete_property', methods=['DELETE'])
@jwt_required()
@limiter.limit("3 per hour")
def delete_property():
    """
    Endpoint to permanently delete a property.
    Requires manual confirmation of the property name.
    """
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /delete_property Deleting property.")

    try:
        # Get the tenant_id from the JWT
        current_user_id = get_jwt_identity()
        tenant_id = get_tenant_from_jwt()

        # Parse request data
        data = request.get_json()
        property_id = data.get('property_id')
        confirm_name = data.get('confirm_name')

        # Input validation
        if not property_id or not confirm_name:
            return utils.jsonify_return_error("error", 400, "Missing required fields."), 400

        # Retrieve property
        property_instance = Property.query.filter_by(property_id=property_id, tenant_id=tenant_id).first()

        if property_instance is None:
            current_app.logger.info(f"{user_ip} - /delete_property Property not found.")
            return utils.jsonify_return_error("error", 404, "Property not found."), 404

        # Check if the provided name matches the actual property name
        if property_instance.name != confirm_name:
            current_app.logger.info(f"{user_ip} - /delete_property Incorrect confirmation name.")
            return utils.jsonify_return_error("error", 400, "Property name does not match. Please enter the correct name to delete."), 400

        # Proceed with hard delete
        db.session.delete(property_instance)
        db.session.commit()

        # Send confirmation email (optional)
        response = utils.send_generic_email(
            current_user_id, 
            "Property Deletion Confirmation",
            f"The property '{property_instance.name}' has been permanently deleted."
        )
        if response.status_code == 404:
            callback_refresh()
            response = utils.send_generic_email(
                current_user_id, 
                "Property Deletion Confirmation",
                f"The property '{property_instance.name}' has been permanently deleted."
            )

        current_app.logger.info(f"{user_ip} - /delete_property success Deleting property.")
        return utils.jsonify_return_success("success", 200, {"message": "Property deleted successfully."}), 200

    except Exception as e:
        current_app.logger.error(f"{user_ip} - /delete_property Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
```

---

## **Security & Best Practices**
✅ **Tenant Validation**: Ensures that only authorized users can delete properties.  
✅ **Hard Delete Only**: No soft delete—deleted properties are **permanently removed**.  
✅ **Manual Confirmation**: Requires users to type the exact name to prevent accidental deletions.  
✅ **Rate Limiting**: Max **3 delete attempts per hour per user** to prevent abuse.  
✅ **Email Notification (optional)**: Sends a confirmation email after deletion.  

---

## **Next Steps**
1️⃣ **Frontend Integration**: Ensure the UI enforces manual name confirmation before allowing deletion.  
2️⃣ **Logging & Monitoring**: Implement logging with **Sentry** and **Slack alerts** for deleted properties.  
3️⃣ **Rate Limit Analytics**: Track deletion attempts to detect **potential abuse**.  

---

### 🔥 **Final Notes**
- **This is a HARD DELETE.** Once removed, a property **cannot be recovered**.  
- Users are **advised to download the property CSV before deletion**.  
- **No refunds** for deleted properties—they simply won’t be renewed.  
