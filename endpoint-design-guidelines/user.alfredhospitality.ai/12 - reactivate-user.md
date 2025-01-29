
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
@limiter.limit("3 per hour")
@jwt_required()
def reactivate_user():
    """
    Endpoint per riattivare l'utente nel sistema.
    Richiede autenticazione tramite JWT.
    """
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /reactivate_user Reactivating user.")

    try:
        # Ottieni l'identità dell'utente dal refresh token
        current_user_id = get_jwt_identity()

        # verifica dell'utente
        # Esegui una query per trovare l'utente nel database
        current_user = User.query.filter_by(user_id=current_user_id).first()

        if current_user is None:
            current_app.logger.info(f"{user_ip} - /reactivate_user User not found")
            return utils.jsonify_return_error("error", 404, "User not found"), 404

        if current_user.is_active:
            current_app.logger.info(f"{user_ip} - /reactivate_user User is already active.")
            return utils.jsonify_return_error("error", 409, "User is already active."), 409
        
        #reactivate the user
        current_user.is_active = True
        db.session.commit()        

        # Restore chatbot data (Subscription must be renewed separately)
        # restore_chatbot_data(user_id)

        # Send reactivation confirmation email
        response = utils.send_generic_email(current_user.email, "Your account has been reactivated", "Your chatbot data has been restored, but you need to renew your subscription.")
        if response.status_code == 404:
            callback_refresh()
            response = utils.send_generic_email(current_user.email, "Your account has been reactivated", "Your chatbot data has been restored, but you need to renew your subscription.")

        # Restituisci la risposta formattata
        current_app.logger.info(f"{user_ip} - /reactivate_user success reactivating user")
        return utils.jsonify_return_success("success", 200, {"message":"User reactivated successfully. Chatbot data restored. Subscription must be renewed separately."}), 200
    
    except Exception as e:
        # Gestione degli errori imprevisti
        current_app.logger.error(f"{user_ip} - /reactivate_user Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
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
