
# **Endpoint: `/refresh_token`**

## **1. Details**
- **URL**: `/refresh_token`
- **Method**: `POST`
- **Authentication**: Requires a valid refresh token.
- **Purpose**: Provide a new access token when the previous one has expired.

---

## **2. Behavior**

### **1. Receiving the Refresh Token**
- Retrieve the refresh token from the `Authorization` header or an HTTP-only cookie.

### **2. Validating the Refresh Token**
- Decode the refresh token using the server's secret key.
- Verify:
  - **Token validity** (not expired).
  - **Token type** (`refresh`).
  - **User existence** in the database.
  - **User status (`is_active`)** → *If `is_active = False`, block the request and return `403 Forbidden`*.

### **3. Generating a New Access Token**
- Create a new access token with a short expiration time.

### **4. Responding to the Frontend**
- Return the new access token.

---

## **3. Accepted Parameters**
- No parameters in the request body.
- Requires the refresh token in the `Authorization` header.

#### Example header:
```
Authorization: Bearer <refresh_token>
```

---

## **4. Endpoint Responses**

### ✅ **Success**
| **HTTP Status** | **Message**                                  |
|----------------|----------------------------------------------|
| `200 OK`       | "Access token refreshed successfully."       |

#### Example Response:
```
{
  "status": "success",
  "code": 200,
  "data": {
    "access_token": "new_access_token"
  }
}
```

---

### ❌ **Errors**

#### 1️⃣ **Missing Refresh Token**
| **HTTP Status** | **Message**                          |
|----------------|--------------------------------------|
| `401 Unauthorized` | "Refresh token missing."         |

#### Example Response:
```
{
  "status": "error",
  "code": 401,
  "message": "Refresh token missing."
}
```

---

#### 2️⃣ **Invalid Refresh Token**
| **HTTP Status** | **Message**                          |
|----------------|--------------------------------------|
| `401 Unauthorized` | "Invalid refresh token."         |

#### Example Response:
```
{
  "status": "error",
  "code": 401,
  "message": "Invalid refresh token."
}
```

---

#### 3️⃣ **User Not Found**
| **HTTP Status** | **Message**                          |
|----------------|--------------------------------------|
| `404 Not Found` | "User not found."                  |

#### Example Response:
```
{
  "status": "error",
  "code": 404,
  "message": "User not found."
}
```

---

#### 4️⃣ **User Inactive (`is_active = False`)**
| **HTTP Status** | **Message**                          |
|----------------|--------------------------------------|
| `403 Forbidden` | "Account is inactive. Please contact support." |

#### Example Response:
```
{
  "status": "error",
  "code": 403,
  "message": "Account is inactive. Please contact support."
}
```

---

#### 5️⃣ **Refresh Token Expired**
| **HTTP Status** | **Message**                          |
|----------------|--------------------------------------|
| `401 Unauthorized` | "Refresh token has expired."     |

#### Example Response:
```
{
  "status": "error",
  "code": 401,
  "message": "Refresh token has expired."
}
```

---

## **5. Updated Code Implementation**

```
@v1.route('/refresh_token', methods=['POST'])
@jwt_required(refresh=True)
def refresh():
    """
    Endpoint to regenerate an access token using a valid refresh token.
    """

    try:
        # Extract user ID from refresh token
        current_user_id = get_jwt_identity()

        # Retrieve user from the database
        user = User.query.filter_by(user_id=current_user_id).first()

        # Check if user exists
        if not user:
            return jsonify_return_error("error", 404, "User not found."), 404

        # Check if user is active
        if not user.is_active:
            return jsonify_return_error("error", 403, "Account is inactive. Please contact support."), 403

        # Generate a new access token
        new_access_token = utils.create_access_token(identity=current_user_id)

        # Prepare response data
        data = {
            "message": "Access token refreshed successfully",
            "access_token": new_access_token
        }

        # Return formatted response
        return utils.jsonify_return_success("success", 200, data), 200

    except ExpiredSignatureError:
        return jsonify_return_error("error", 401, "Refresh token has expired."), 401

    except Exception as e:
        return jsonify_return_error("error", 500, "An unexpected error occurred."), 500
```

---

## **6. Validations to Implement**

1️⃣ **JWT Token Validation**
- Ensure the token is valid and not expired.
- Verify that the token type is `refresh`.

2️⃣ **User Status Check (`is_active`)**
- If `is_active = False`, **block token refresh** and return `403 Forbidden`.

3️⃣ **Blacklist Implementation**
- Implement a refresh token blacklist for logout scenarios.

4️⃣ **Refresh Token Rotation (optional improvement)**
- Issue a new refresh token when the current one is used.

5️⃣ **Advanced Security Enhancements (optional improvement)**
- Include **`jti` (unique identifier)** to track token usage.

---

## **7. Next Steps**
✅ **Update frontend to handle `403 Forbidden` errors properly**  
✅ **Implement token blacklist and refresh rotation (optional)**  
✅ **Test cases:**  
   - Valid token refresh  
   - Expired refresh token  
   - Invalid refresh token  
   - User not found  
   - User inactive (`is_active = False`)  

---
