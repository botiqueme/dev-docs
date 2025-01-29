# **Endpoint: `/activate_chatbot/{property_id}`**

## **Purpose**
This endpoint allows a user to **activate** the chatbot for a specific property.  
- **Activation** is only possible if the user has an **active subscription**.  
- **Once activated, the chatbot becomes available on supported platforms (e.g., WhatsApp, Telegram).**  
- **Rate Limiting**: Prevents excessive activation attempts (1 attempt per minute per property).  

---

## **Technical Specifications**

### **Method**
`PATCH`

### **URL**
`/activate_chatbot/{property_id}`

### **Authentication**
🔑 **Requires JWT Access Token**

---

## **Request Parameters**

| **Parameter**  | **Type**  | **Required** | **Description** |
|---------------|-----------|--------------|-----------------|
| `property_id` | UUID      | ✅ Yes       | The unique identifier of the property. |

**Example Request:**
```
PATCH /activate_chatbot/123e4567-e89b-12d3-a456-426614174000
Authorization: Bearer <JWT_TOKEN>
```

---

## **Response Codes**

| **Scenario** | **HTTP Status** | **Message** |
|-------------|----------------|-------------|
| ✅ **Success** | `200 OK` | `"Chatbot activated successfully."` |
| ❌ **Property Not Found** | `404 Not Found` | `"Property not found."` |
| ❌ **User Not Authorized** | `403 Forbidden` | `"Unauthorized to modify this property."` |
| ❌ **Subscription Inactive** | `403 Forbidden` | `"An active subscription is required to activate the chatbot."` |
| ❌ **Already Activated** | `409 Conflict` | `"Chatbot is already active."` |
| ❌ **Rate Limit Exceeded** | `429 Too Many Requests` | `"Too many activation attempts. Please wait before trying again."` |

---

## **Implementation Logic**

### **1. Authentication & Authorization**
- Verify **JWT Token**.
- Extract **user_id** from the token.
- Retrieve **property** and check if the user owns it.

### **2. Subscription Validation**
- Ensure the user has an **active subscription** before activating the chatbot.

### **3. Prevent Repeated Activation**
- If the chatbot is **already active**, return **409 Conflict**.

### **4. Activate Chatbot**
- Set `chatbot_active = True`.
- **Log the activation** for tracking purposes.
- **Send an email notification** confirming activation.

### **5. Enforce Rate Limiting**
- **1 activation attempt per minute per property** to prevent spam.
- If exceeded, return `429 Too Many Requests`.

---

## **Implementation Code**
```python
@v1.route('/activate_chatbot/<uuid:property_id>', methods=['PATCH'])
@limiter.limit("1 per minute", key_func=lambda: f"{get_jwt_identity()}-{request.view_args['property_id']}")
@jwt_required()
def activate_chatbot(property_id):
    """
    Activate chatbot for a property.
    Requires JWT authentication.
    Rate-limited to 1 activation per minute per property.
    """
    user_ip = utils.get_client_ip(request)
    current_user_id = get_jwt_identity()
    current_app.logger.info(f"{user_ip} - /activate_chatbot Attempting activation for property {property_id}")

    # Retrieve property
    property = Property.query.filter_by(property_id=property_id, user_id=current_user_id).first()
    
    if not property:
        current_app.logger.info(f"{user_ip} - /activate_chatbot Property not found.")
        return utils.jsonify_return_error("error", 404, "Property not found."), 404

    # Check if chatbot is already active
    if property.chatbot_active:
        current_app.logger.info(f"{user_ip} - /activate_chatbot Chatbot is already active.")
        return utils.jsonify_return_error("error", 409, "Chatbot is already active."), 409

    # Ensure user has an active subscription
    if not utils.has_active_subscription(current_user_id):
        current_app.logger.info(f"{user_ip} - /activate_chatbot Subscription inactive, cannot activate.")
        return utils.jsonify_return_error("error", 403, "An active subscription is required to activate the chatbot."), 403

    # Activate chatbot
    property.chatbot_active = True
    db.session.commit()

    # Log action
    current_app.logger.info(f"{user_ip} - /activate_chatbot Chatbot activated for property {property_id}")

    # Send confirmation email
    response = utils.send_generic_email(
        property.user.email,
        "Your chatbot has been activated",
        f"The chatbot for your property **{property.name}** has been successfully activated."
    )

    if response.status_code == 404:
        callback_refresh()
        utils.send_generic_email(
            property.user.email,
            "Your chatbot has been activated",
            f"The chatbot for your property **{property.name}** has been successfully activated."
        )

    return utils.jsonify_return_success("success", 200, {"message": "Chatbot activated successfully."}), 200
```

---

## **Security & Performance**
✔ **JWT Authentication** → Ensures only the property owner can activate the chatbot.  
✔ **Subscription Check** → Prevents unpaid activations.  
✔ **Rate Limiting (1 activation/min per property)** → Prevents excessive activation attempts.  
✔ **Audit Logging** → Logs all activations for security tracking.  
✔ **Email Notification** → Confirms activation to the user.  

