
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

    # Check if the user is active
    if not user.is_active:
        return jsonify_return_error("error", 403, "User account is inactive. Please contact support."), 403

    # Return user data
    return jsonify_return_success("success", 200, {
        "user_id": user.user_id,
        "email": user.email,
        "name": user.name,
        "surname": user.surname,
        "phone_number": user.phone_number,
        "company": user.company,
        "vat_number": user.vat_number,
        "is_verified": user.is_verified,
        "is_active": user.is_active
    })
```

---

## 6. Security Considerations

1. **Authentication**
   - JWT tokens should be securely stored and validated.

2. **Inactive Users Are Blocked**
   - Ensures inactive users cannot retrieve their profile.

3. **Rate Limiting**
   - Optional rate limiting to prevent abuse.



