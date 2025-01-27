### **Endpoint: `/deactivate_chatbot/{property_id}`**

## **Purpose**
This endpoint allows a user to deactivate the chatbot for a specific property. Deactivation **does not delete logs or past chatbot interactions**; it simply disables the chatbot's availability on platforms such as **WhatsApp, Telegram, and web chat**.  

A **notification** will be sent to confirm the deactivation.  
All operations are **logged** for auditing purposes.

---

## **Technical Specifications**

### **Method**
`PATCH`

### **URL**
`/deactivate_chatbot/{property_id}`

### **Authentication**
🔑 Requires **JWT Token (Access Level)**

### **Rate Limiting**
⏳ **5 requests per minute per user** to prevent abuse.

---

## **Request Parameters**

| **Parameter**  | **Type**   | **Required** | **Description** |
|---------------|------------|--------------|-----------------|
| `property_id` | Path Param | ✅ Yes       | The ID of the property whose chatbot should be deactivated. |
| `Authorization` | Header (JWT) | ✅ Yes  | Bearer token to authenticate the user. |

**Example Request:**
```
PATCH /deactivate_chatbot/123e4567-e89b-12d3-a456-426614174000
Authorization: Bearer <JWT_TOKEN>
```

---

## **Response Codes**

| **Scenario**              | **HTTP Status** | **Message** |
|--------------------------|----------------|-------------|
| ✅ **Success** | `200 OK` | `"Chatbot deactivated successfully."` |
| ❌ **Property Not Found** | `404 Not Found` | `"Property not found."` |
| ❌ **Chatbot Already Deactivated** | `409 Conflict` | `"Chatbot is already deactivated for this property."` |
| ❌ **Unauthorized Access** | `403 Forbidden` | `"You do not have permission to modify this property."` |
| ⏳ **Rate Limit Exceeded** | `429 Too Many Requests` | `"Too many deactivation attempts. Please try again later."` |

---

## **Implementation Logic**

### **1. Authentication & Authorization**
- Extract the JWT token from the `Authorization` header.
- Decode the token and retrieve the **user ID**.
- Validate that the user has **access** to the property.

### **2. Property & Chatbot Status Verification**
- Retrieve the property using `property_id`.
- If the property does not exist, return **404 Not Found**.
- If the chatbot is **already deactivated**, return **409 Conflict**.

### **3. Deactivation Process**
- **Set `is_chatbot_active = False`** in the database.
- Ensure that **logs and past interactions remain intact**.

### **4. Logging & Auditing**
- Log:
  - `user_id`
  - `property_id`
  - `timestamp`
  - `previous chatbot status`
  - `new chatbot status`
- Logs should be stored in **an audit table or monitoring system**.

### **5. Send Notification**
- **Notify the user via email or in-app notification** confirming the chatbot's deactivation.

---

## **Implementation Code**
```python
@v1.route('/deactivate_chatbot/<property_id>', methods=['PATCH'])
@limiter.limit("5 per minute")
@jwt_required()
def deactivate_chatbot(property_id):
    """
    Endpoint to deactivate the chatbot for a property.
    The chatbot logs remain, but it is no longer active.
    """
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /deactivate_chatbot Deactivating chatbot for property {property_id}.")

    try:
        # Extract user ID from JWT
        current_user_id = get_jwt_identity()

        # Retrieve the property
        property_obj = Property.query.filter_by(property_id=property_id, user_id=current_user_id).first()

        if not property_obj:
            current_app.logger.info(f"{user_ip} - /deactivate_chatbot Property not found.")
            return utils.jsonify_return_error("error", 404, "Property not found"), 404

        # Check if chatbot is already deactivated
        if not property_obj.is_chatbot_active:
            current_app.logger.info(f"{user_ip} - /deactivate_chatbot Chatbot already deactivated.")
            return utils.jsonify_return_error("error", 409, "Chatbot is already deactivated for this property."), 409

        # Deactivate chatbot
        property_obj.is_chatbot_active = False
        db.session.commit()

        # Log the operation
        utils.log_audit_event(
            user_id=current_user_id,
            property_id=property_id,
            action="CHATBOT_DEACTIVATION",
            status="SUCCESS"
        )

        # Send confirmation notification
        utils.send_notification(current_user_id, "Chatbot deactivated for your property.", "success")

        current_app.logger.info(f"{user_ip} - /deactivate_chatbot Chatbot deactivated successfully.")
        return utils.jsonify_return_success("success", 200, {"message": "Chatbot deactivated successfully."}), 200

    except Exception as e:
        current_app.logger.error(f"{user_ip} - /deactivate_chatbot Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
```

---

## **Security & Best Practices**
✅ **Rate Limiting:** Prevents abuse (5 requests per minute per user).  
✅ **JWT Authentication:** Ensures only authorized users can deactivate chatbots.  
✅ **Logging & Monitoring:** Tracks changes for security audits.  
✅ **In-App Notifications:** Notifies users about the status change.  

---

## **Next Steps**
1. **Frontend Handling:** Ensure the UI reflects chatbot deactivation instantly.  
2. **Add User Notifications:** Allow users to **opt-in** for email confirmations.  
3. **Monitoring:** Integrate **Sentry** for error tracking & **Slack notifications** for system monitoring.  

