
# **Endpoint: `/reactivate_user`**

## **Purpose**
This endpoint allows users to **reactivate** their accounts if they were previously deactivated. **Reactivation restores chatbot data** but does **not** automatically renew the user's subscription.

---

## **Technical Specifications**

### **Method**
`PATCH`

### **URL**
`/reactivate_user`

### **Authentication**
**JWT Token Required**

---

## **Request Parameters**

| **Parameter**  | **Type**  | **Required** | **Description** |
|---------------|-----------|--------------|-----------------|
| `Authorization` | Header (JWT) | yes | Bearer token to authenticate the user |

**Example Request:**
```
PATCH /reactivate_user
Authorization: Bearer <JWT_TOKEN>
```

---

## **Response Codes**

| **Scenario** | **HTTP Status** | **Message** |
|-------------|----------------|-------------|
|  **Success** | `200 OK` | `"User reactivated successfully. Chatbot data restored. Subscription must be renewed separately."` |
|  **User Not Found** | `404 Not Found` | `"User not found."` |
|  **Already Active** | `409 Conflict` | `"User is already active."` |
|  **Rate Limit Exceeded** | `429 Too Many Requests` | `"Too many reactivation attempts. Please try again later."` |

---

## **Implementation Logic**

### **1. JWT Authentication**
- Extract the JWT token from the `Authorization` header.
- Decode the token and retrieve the **user ID**.
- Validate that the user **exists** in the database.

### **2. Validate User Status**
- If `is_active = True`, return **409 Conflict** (`"User is already active."`).
- If `is_active = False`, proceed with reactivation.

### **3. Reactivation Process**
- Set `is_active = True` in the database.
- Restore **chatbot data** but **do not renew the subscription**.
- Update `reactivation_timestamp` for audit purposes.

### **4. Logging & Monitoring**
- Log the reactivation attempt (`user_id`, timestamp, and IP address).
- Track **successful and failed** attempts.

### **5. Reactivation Confirmation Email**
- Send an **email notification** confirming account reactivation.

---

## **Implementation Code**
```
@v1.route('/reactivate_user', methods=['PATCH'])
@limiter.limit("3 per hour")  # Rate limiting: 3 attempts per hour
def reactivate_user():
    # Extract JWT token
    auth_header = request.headers.get('Authorization')
    if not auth_header:
        return jsonify({"status": "error", "code": 401, "message": "Authorization token required."}), 401

    try:
        token = auth_header.split(" ")[1]
        payload = jwt.decode(token, current_app.config['SECRET_KEY'], algorithms=['HS256'])
        user_id = payload['user_id']
    except Exception:
        return jsonify({"status": "error", "code": 401, "message": "Invalid token."}), 401

    # Retrieve user from the database
    user = User.query.filter_by(user_id=user_id).first()
    if not user:
        return jsonify({"status": "error", "code": 404, "message": "User not found."}), 404

    # Check if the user is already active
    if user.is_active:
        return jsonify({"status": "error", "code": 409, "message": "User is already active."}), 409

    # Reactivate the user
    user.is_active = True
    user.reactivation_timestamp = datetime.utcnow()
    db.session.commit()

    # Restore chatbot data (Subscription must be renewed separately)
    restore_chatbot_data(user_id)

    # Logging reactivation attempt
    logger.info(f"User {user_id} reactivated at {user.reactivation_timestamp}")

    # Send reactivation confirmation email
    send_email(user.email, "Your account has been reactivated", "Your chatbot data has been restored, but you need to renew your subscription.")

    return jsonify({
        "status": "success",
        "code": 200,
        "message": "User reactivated successfully. Chatbot data restored. Subscription must be renewed separately."
    }), 200
```

---

## **Security & Best Practices**
 **Rate Limiting**: **3 reactivation attempts per hour per user**.  
 **Logging & Monitoring**: Logs stored for **security audits** and **analytics**.  
 **Audit Trail**: Stores **deactivation & reactivation timestamps**.  
 **Email Notification**: User receives a **confirmation email upon reactivation**.  
 **Ensures GDPR Compliance**: Keeps **user control over data reactivation**.  

---

## **Next Steps**
**Frontend Handling:** Ensure the UI informs users they must **renew their subscription** separately.  
