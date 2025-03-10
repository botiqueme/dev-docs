
# **Endpoint: `/upload_profile_picture`**

## **Purpose**
This endpoint allows users to **upload** their accounts profile picture.

---

## **Technical Specifications**

### **Method**
`POST`

### **URL**
`/upload_profile_picture`

### **Authentication**
**JWT Token Required**

---

## **Request Parameters**

| **Parameter**  | **Type**  | **Required** | **Description** |
|---------------|-----------|--------------|-----------------|
| `Authorization` | Header (JWT) | yes | Bearer token to authenticate the user |

**Example Request:**
```
PUT /upload_profile_picture
Authorization: Bearer <JWT_TOKEN>
```

---

## **Response Codes**

| **Scenario** | **HTTP Status** | **Message** |
|-------------|----------------|-------------|
|  **Success** | `200 OK` | `"Profile picture uploaded successfully"` |
|  **User Not Found** | `404 Not Found` | `"User not found."` |
|  **Image not provided** | `400 Conflict` | `"Image not provided."` |
|  **Rate Limit Exceeded** | `429 Too Many Requests` | `"Too many reactivation attempts. Please try again later."` |

---

## **Implementation Logic**

### **1. JWT Authentication**
- Extract the JWT token from the `Authorization` header.
- Decode the token and retrieve the **user ID**.
- Validate that the user **exists** in the database.

### **2. Upload profile picture Process**

### **3. Upload profile picture URL in the user database**

---

## **Implementation Code**
```
@v1.route('/upload_profile_picture', methods=['POST'])
@limiter.limit("3 per hour")
@jwt_required()
def upload_profile_picture_endpoint():
    """
    Endpoint per caricare l'immagine del profilo dell'utente.
    Richiede autenticazione tramite JWT.
    """
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /upload_profile_picture Upload profile picture")
    
    try:
        # Ottieni l'identità dell'utente dal refresh token
        current_user_id = UUID(get_jwt_identity())

        # verifica dell'utente
        # Esegui una query per trovare l'utente nel database
        current_user = User.query.filter_by(user_id=current_user_id).first()

        if current_user is None:
            current_app.logger.info(f"{user_ip} - /upload_profile_picture User not found")
            return utils.jsonify_return_error("error", 404, "User not found"), 404
        
        if 'profile_picture' not in request.files:
            return utils.jsonify_return_error("error", 400, "/upload_profile_picture No file provided"), 400
    
        file = request.files['profile_picture']

        # Carica l'immagine su Azure e ottieni URL con SAS Token
        try:
            blob_url = utils.upload_profile_picture(blob_service_client, file, str(current_user_id))
        
            if blob_url:
                # Salva l'URL dell'immagine del profilo nel database
                current_user.profile_picture_url = blob_url
                db.session.commit()
                
                current_app.logger.info(f"{user_ip} - /upload_profile_picture Profile picture uploaded successfully")
                return utils.jsonify_return_success("success", 200, {"url": blob_url}), 200

            else:
                current_app.logger.error(f"{user_ip} - /upload_profile_picture Internal Server Error. {blob_url}")
                return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
        except Exception as e:
            current_app.logger.error(f"{user_ip} - /upload_profile_picture Internal Server Error. {e}")
            return utils.jsonify_return_error("error", 500, "Internal Server."), 500
        
    except Exception as e:
        # Gestione degli errori imprevisti
        current_app.logger.error(f"{user_ip} - /upload_profile_picture Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500

```
