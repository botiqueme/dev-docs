
# **Endpoint: `/get_profile_picture`**

## **Purpose**
This endpoint allows frontend to **get** the user accounts profile picture.

---

## **Technical Specifications**

### **Method**
`GET`

### **URL**
`/get_profile_picture`

### **Authentication**
**JWT Token Required**

---

## **Request Parameters**

| **Parameter**  | **Type**  | **Required** | **Description** |
|---------------|-----------|--------------|-----------------|
| `Authorization` | Header (JWT) | yes | Bearer token to authenticate the user |

**Example Request:**
```
PUT /get_profile_picture
Authorization: Bearer <JWT_TOKEN>
```

---

## **Response Codes**

| **Scenario** | **HTTP Status** | **Message** |
|-------------|----------------|-------------|
|  **Success** | `200 OK` | `"Profile picture uploaded successfully"` |
|  **User Not Found** | `404 Not Found` | `"User not found."` |
|  **Rate Limit Exceeded** | `429 Too Many Requests` | `"Too many reactivation attempts. Please try again later."` |

---

## **Implementation Logic**

### **1. JWT Authentication**
- Extract the JWT token from the `Authorization` header.
- Decode the token and retrieve the **user ID**.
- Validate that the user **exists** in the database.

### **2. Retrieve profile picture URL**

---

## **Implementation Code**
```
@v1.route('/get_profile_picture', methods=['GET'])
@jwt_required()
def get_profile_picture():
    """
    Endpoint per ottenere l'immagine del profilo dell'utente.
    Richiede autenticazione tramite JWT.
    """
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /get_profile_picture Getting profile picture.")
    
    try:
        # Ottieni l'identità dell'utente dal refresh token
        current_user_id = UUID(get_jwt_identity())

        # verifica dell'utente
        # Esegui una query per trovare l'utente nel database
        current_user = User.query.filter_by(user_id=current_user_id).first()

        if current_user is None:
            current_app.logger.info(f"{user_ip} - /get_profile_picture User not found")
            return utils.jsonify_return_error("error", 404, "User not found"), 404
        
        # Restituisci l'URL dell'immagine del profilo
        current_app.logger.info(f"{user_ip} - /get_profile_picture Profile picture found")
        return utils.jsonify_return_success("success", 200, {"url": current_user.profile_picture_url}), 200
    
    except Exception as e:
        # Gestione degli errori imprevisti
        current_app.logger.error(f"{user_ip} - /get_profile_picture Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
```
