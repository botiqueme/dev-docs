# Endpoint: `/get_user`

## Purpose
This endpoint retrieves the details of an authenticated user, allowing users to view their profile information.

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

There are no parameters in the body of the request.

### **Request Headers**
- `Authorization`: Bearer token (JWT)

E.g., request header:
```
Authorization: Bearer <jwt_token>
```

---

## 3. Logic

### **1. Authentication**
- Retrieve the JWT from the `Authorization` header.
- Decode the JWT to obtain the `user_id`.
- Verify that the user exists in the database and is authorized to access their profile.

### **2. Data Retrieval**
- Fetch the user's data from the database based on the `user_id`.

### **3. Soft Delete/Deactivation Logic**
- If the user has deactivated their account, return the user data with a flag indicating the account is deactivated, and inform the user about the status (if necessary).
- If the user's data is pending deletion after 6 months, return a message indicating that the data is no longer available.

### **4. Response**
- Return the user’s profile information or an error message if the user is not found or unauthorized.

---

## 4. Responses

### **Success**
| **HTTP Status** | **Message**                           |
|-----------------|---------------------------------------|
| `200 OK`        | User data successfully retrieved.     |

Response Body Example:
```
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
    "deactivated": false  // Optional: if applicable
  }
}
```

---

### **Error: User Not Found**
| **HTTP Status** | **Message**                           |
|-----------------|---------------------------------------|
| `404 Not Found` | User not found.                       |

Response Body Example:
```
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

Response Body Example:
```
{
  "status": "error",
  "code": 401,
  "message": "Authorization token required."
}
```

---

## 5. Code Implementation

```
@v1.route('/get_user', methods=['GET'])
def get_user():
    # Authentication via JWT
    auth_header = request.headers.get('Authorization')
    if not auth_header:
        return jsonify_return_error("error", 401, "Authorization token required"), 401

    try:
        token = auth_header.split(" ")[1]
        payload = jwt.decode(token, current_app.config['SECRET_KEY'], algorithms=['HS256'])
        user_id = payload['user_id']
    except Exception:
        return jsonify_return_error("error", 401, "Invalid token"), 401

    # Retrieve user from database
    user = User.query.filter_by(user_id=user_id).first()
    if not user:
        return jsonify_return_error("error", 404, "User not found"), 404

    # Response with user data
    return jsonify_return_success("success", 200, {
        "user_id": user.user_id,
        "email": user.email,
        "name": user.name,
        "surname": user.surname,
        "phone_number": user.phone_number,
        "company": user.company,
        "vat_number": user.vat_number,
        "is_verified": user.is_verified,
        "deactivated": user.deactivated  # Optional: if applicable
    })
```

---

## 6. Security Considerations

1. **Authentication**:
   - JWT tokens should be securely stored and validated.
2. **Sensitive Data Protection**:
   - Ensure that sensitive data such as passwords and personal details are properly encrypted.
3. **Rate Limiting**:
   - Rate limiting can be applied to prevent abuse of the endpoint.
