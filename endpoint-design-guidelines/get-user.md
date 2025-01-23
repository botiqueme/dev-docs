
# Endpoint: `/get_user`

## Purpose
This endpoint retrieves the details of an authenticated user, allowing them to view their profile information.

---

## 1. Technical Details

### **Method**
`GET`

### **URL**
`/get_user`

### **Authentication**
Requires a valid JWT token to identify and authorize the user.

---

## 2. Parameters

No request body parameters.

### **Request Headers**
- `Authorization`: Bearer token (JWT)

**Example Request Header:**
```
Authorization: Bearer <jwt_token>
```

---

## 3. Logic

### **1. Authentication**
- Retrieve the JWT from the `Authorization` header.
- Decode the JWT to obtain the `user_id`.
- Verify that the user exists in the database and is authorized.

### **2. `is_active` Check**
- If `is_active = False`, return **403 Forbidden** and prevent profile access.

### **3. Data Retrieval**
- Fetch the user’s profile from the database.

### **4. Response**
- If active, return the user’s profile.
- If inactive, deny access.

---

## 4. Responses

### **Success**
| **HTTP Status** | **Message**                           |
|-----------------|---------------------------------------|
| `200 OK`        | User data successfully retrieved.     |

**Response Example:**
```json
{
  "status": "success",
  "code": 200,
  "data": {
    "user_id": "123e4567-e89b-12d3-a456-426614174000",
    "email": "user@example.com",
    "name": "John",
    "surname": "Doe",
    "phone_number": "+1234567890",
    "company": "Example Corp",
    "vat_number": "IT12345678901",
    "is_verified": true,
    "is_active": true
  }
}
```

---

### **Error: Inactive User**
| **HTTP Status** | **Message**                           |
|-----------------|---------------------------------------|
| `403 Forbidden` | User account is inactive.             |

**Response Example:**
```json
{
  "status": "error",
  "code": 403,
  "message": "User account is inactive. Please contact support."
}
```

---

### **Error: User Not Found**
| **HTTP Status** | **Message**                           |
|-----------------|---------------------------------------|
| `404 Not Found` | User not found.                       |

**Response Example:**
```json
{
  "status": "error",
  "code": 404,
  "message": "User not found."
}
```

---

### **Error: Unauthorized**
| **HTTP Status** | **Message**                           |
|-----------------|---------------------------------------|
| `401 Unauthorized` | Unauthorized access.               |

**Response Example:**
```json
{
  "status": "error",
  "code": 401,
  "message": "Authorization token required."
}
```

---

## 5. Code Implementation

```python
@v1.route('/get_user', methods=['GET'])
@jwt_required()
def get_user():
    """
    Endpoint per ottenere le informazioni dell'utente.
    Richiede autenticazione tramite JWT.
    """
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /get_user Updating user info.")

    try:
        # Ottieni l'identità dell'utente dal refresh token
        current_user_id = get_jwt_identity()

        # verifica dell'utente
        # Esegui una query per trovare l'utente nel database
        current_user = User.query.filter_by(user_id=current_user_id).first()

        if current_user is None:
            current_app.logger.info(f"{user_ip} - /get_user User not found")
            return utils.jsonify_return_error("error", 404, "User not found"), 404
        if not current_user.is_active:
            current_app.logger.info(f"{user_ip} - /get_user Account is inactive. Please contact support.")
            return utils.jsonify_return_error("error", 403, "Account is inactive. Please contact support."), 403
        
            # Prepara i dati della risposta
        data = {
            "user_id": current_user.user_id,
            "email": current_user.email,
            "name": current_user.name,
            "surname": current_user.surname,
            "phone_number": current_user.phone_number,
            "vat_number": current_user.vat_number,
            "is_verified": current_user.is_verified,
            "is_active": current_user.is_active
        }

        # Restituisci la risposta formattata
        current_app.logger.info(f"{user_ip} - /get_user success refreshing token")
        return utils.jsonify_return_success("success", 200, data), 200
    
    except Exception as e:
        # Gestione degli errori imprevisti
        current_app.logger.error(f"{user_ip} - /get_user Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
```

---

## 6. Security Considerations

1. **Authentication**
   - JWT tokens should be securely stored and validated.

2. **Inactive Users Are Blocked**
   - Ensures inactive users cannot retrieve their profile.

3. **Rate Limiting**
   - Optional rate limiting to prevent abuse.



