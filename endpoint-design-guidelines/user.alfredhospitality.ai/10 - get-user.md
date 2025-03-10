

# **Endpoint: `/get_user`**

## **Purpose**
Retrieves the details of an authenticated user.  

## **1. Technical Specifications**

### **Method**
`GET`

### **URL**
`/get_user`

### **Authentication**
✅ Requires a valid JWT token (Bearer Authentication).  
---

### **Request Headers**
| **Header**       | **Required** | **Description**               |
|------------------|-------------|-------------------------------|
| `Authorization`  | ✅ Yes       | JWT Token for authentication. |

**Example Request (Standard User)**
```
GET /get_user
Authorization: Bearer <JWT_TOKEN>
```
---

## **3. Flow Logic**

1️⃣ **JWT Authentication**
   - Extract `Authorization` token.
   - Validate token & extract `user_id`.

2️⃣ **Retrieve User from Database**
   - Fetch user details using `user_id`.

3️⃣ **Check `is_active` Status**
   - ✅ ** Users**  
      - If `is_active = False`, return `403 Forbidden`.  

4️⃣ **Format Response**
   - Include **all relevant fields**:
     - **Personal Users** → `name, surname, email, phone_number, billing_address`
     - **Company Users** → Additional fields: `company_name, company_vat, company_billing_address, company_phone_number, job_title`

5️⃣ **Return Success Response**  
   - `200 OK` → User data is returned.  
   - `403 Forbidden` → User is inactive & not an admin.  
   - `404 Not Found` → User does not exist.  

---

## **4. Responses**

### ✅ **Success**
| **HTTP Status** | **Message**                           |
|-----------------|---------------------------------------|
| `200 OK`        | User data successfully retrieved.     |

**Response Example (Personal User)**
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
    "billing_address": "123 Main St",
    "is_company": false,
    "is_verified": true,
    "is_active": true
  }
}
```

**Response Example (Company User)**
```json
{
  "status": "success",
  "code": 200,
  "data": {
    "user_id": "456e7890-a12b-34c5-d678-901234567890",
    "email": "ceo@business.com",
    "name": "Jane",
    "surname": "Smith",
    "phone_number": null,
    "billing_address": null,
    "is_company": true,
    "company_name": "TechCorp Ltd.",
    "company_vat": "IT987654321",
    "company_billing_address": "456 Business Rd",
    "company_phone_number": "+9876543210",
    "job_title": "CEO",
    "is_verified": true,
    "is_active": true
  }
}
```

---

### ❌ **Errors**
| **Cause**                  | **HTTP Status** | **Message**                                  |
|----------------------------|----------------|----------------------------------------------|
| User not found             | `404 Not Found` | `"User not found."`                         |
| Account is inactive        | `403 Forbidden` | `"Account is inactive. Please reactivate."` |
| Unauthorized (No Token)    | `401 Unauthorized` | `"Authorization token required."`          |

**Example Error (Inactive User, Non-Admin)**
```json
{
  "status": "error",
  "code": 403,
  "message": "Account is inactive. Please reactivate your account."
}
```

---

## **5. Updated Code Implementation**

```python
@v1.route('/get_user', methods=['GET'])
@jwt_required()
def get_user():
    """
    Endpoint per ottenere le informazioni dell'utente.
    Richiede autenticazione tramite JWT.
    """
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /get_user Getting user info.")

    try:
        # Ottieni l'identità dell'utente dal refresh token
        current_user_id = UUID(get_jwt_identity())

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
            "billing_address": current_user.billing_address,
            "company_name": current_user.company_name,
            "company_billing_address": current_user.company_billing_address,
            "company_phone_number": current_user.company_phone_number,
            "job_title": current_user.job_title,
            "is_company": current_user.is_company,
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

## **6. Security Considerations**

✅ **Authentication**
   - JWT tokens **must be securely validated** before accessing user data.

✅ **Restricting Inactive Users**
   - Inactive users **cannot** access their account.

✅ **Rate Limiting**
   - Prevent abuse by **limiting requests per minute**.

---

## **7. Database Considerations**
1. **Ensure `is_active` Is Checked on Login**
   - Users should **not** be able to log in if inactive.
2. **Ensure Unique Company VAT**
   - The VAT number must be **unique per company**.

---

## **8. Future Considerations**
🔹 **Audit Logs for Admin Requests**  
- Track **when & why** an admin retrieves user data.  
- Store logs in a secure table for **transparency**.

🔹 **User Account Deletion Policy**  
- Decide **whether inactive users should be automatically removed** after X months.

---

### ✅ **Final Thoughts**
With this approach, we **keep everything inside a single endpoint**, but ensure:
- 🔐 **Security** (Admins can retrieve inactive users, but regular users can't).
- 🏗️ **Scalability** (Easily extendable for future use cases).
- ✅ **Consistency** (All user details, both personal & company, are handled properly).  
