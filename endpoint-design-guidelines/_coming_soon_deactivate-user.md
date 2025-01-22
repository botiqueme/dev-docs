# Endpoint: `/deactivate_user`

## Purpose
This endpoint allows administrators or users (depending on the access logic) to temporarily deactivate a user account. By deactivating the account, the user will no longer be able to access the system, but their data will remain in the database. The account can later be reactivated through another endpoint.

---

## Technical Specifications

### **Method**
`PATCH`

### **URL**
`/deactivate_user`

### **Authentication**
Requires a JWT token to identify and authorize the user requesting the deactivation. Alternatively, this endpoint can be restricted to administrators only.

---

## Request Parameters

| **Parameter** | **Type**  | **Required** | **Description**                             |
|---------------|-----------|--------------|---------------------------------------------|
| `user_id`     | String    | Yes          | The ID of the user to deactivate.           |

Example request:
```
{
  "user_id": "123e4567-e89b-12d3-a456-426614174000"
}
```

---

## Endpoint Logic

### 1. **Authentication and Authorization**
- Retrieve the JWT token from the `Authorization` header.
- Decode the token to identify the user.
- If the user is an administrator, proceed to deactivate another account. If the user is not an administrator, they can only deactivate their own account.

### 2. **Validations**
- Check that the `user_id` is present in the request.
- Verify that the specified user exists in the database.
- Ensure the user is not already deactivated.

### 3. **Deactivate Account**
- Update the user's status in the database (e.g., setting the `is_active` field to `False`).
- The account will be deactivated, but the user's data will remain in the system.

### 4. **Responses**
- Return a success status with a message confirming the deactivation or an error if issues occur (e.g., user not found, already deactivated).

---

## Endpoint Responses

### **Success**
| **HTTP Status** | **Message**                         |
|-----------------|--------------------------------------|
| `200 OK`        | "User account deactivated successfully." |

### **Errors**
| **Cause**                | **HTTP Status**   | **Message**                                    |
|--------------------------|-------------------|-------------------------------------------------|
| User not found           | `404 Not Found`   | "User not found."                               |
| Account already deactivated | `409 Conflict`    | "User account already deactivated."             |
| Missing `user_id` parameter | `400 Bad Request` | "Missing user_id."                             |
| Unauthorized             | `403 Forbidden`    | "You do not have permission to deactivate this user." |

---

## Code Implementation

```
from flask import request, jsonify, current_app
from app.models import User
from app import db
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
import logging

# Logging configuration
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Rate limiter configuration
limiter = Limiter(key_func=get_remote_address)

@v1.route('/deactivate_user', methods=['PATCH'])
@limiter.limit("3 per minute")
def deactivate_user():
    auth_header = request.headers.get('Authorization')
    if not auth_header:
        return jsonify({
            "status": "error",
            "code": 401,
            "message": "Authorization token required."
        }), 401

    try:
        token = auth_header.split(" ")[1]
        payload = jwt.decode(token, current_app.config['SECRET_KEY'], algorithms=['HS256'])
        user_id = payload['user_id']
        is_admin = payload.get('is_admin', False)
    except Exception:
        return jsonify({
            "status": "error",
            "code": 401,
            "message": "Invalid token."
        }), 401

    # Check that user_id is present in the request
    data = request.get_json()
    target_user_id = data.get('user_id')
    if not target_user_id:
        return jsonify({
            "status": "error",
            "code": 400,
            "message": "Missing user_id."
        }), 400

    # If not an admin, the user can only deactivate their own account
    if not is_admin and user_id != target_user_id:
        return jsonify({
            "status": "error",
            "code": 403,
            "message": "You do not have permission to deactivate this user."
        }), 403

    # Retrieve user from database
    user = User.query.filter_by(user_id=target_user_id).first()
    if not user:
        return jsonify({
            "status": "error",
            "code": 404,
            "message": "User not found."
        }), 404

    # Check if the user is already deactivated
    if not user.is_active:
        return jsonify({
            "status": "error",
            "code": 409,
            "message": "User account already deactivated."
        }), 409

    # Deactivate the account
    user.is_active = False
    db.session.commit()

    logger.info(f"User {target_user_id} account deactivated successfully.")
    return jsonify({
        "status": "success",
        "code": 200,
        "message": "User account deactivated successfully."
    }), 200
```
---

## Security and Improvements Implemented

1. **Rate Limiting**:
   - Configured with `Flask-Limiter` to prevent abuse and brute-force attacks (3 requests per minute per IP).

2. **JWT Authentication**:
   - Authentication through JWT token to ensure only authorized users can deactivate accounts.

3. **Authorization Check**:
   - Only administrators or the user themselves can deactivate the account.

4. **Advanced Error Handling**:
   - Detailed error handling for various failure scenarios, including user not found and unauthorized deactivation requests.

---

## Suggested Future Improvements

### 1. **Reactivate User Endpoint** (optional)
- Implement an endpoint `/reactivate_user` that allows users or admins to reactivate a previously deactivated account.

### 2. **Advanced Logging** (optional)
- Add more detailed logging to track deactivation operations, including reasons (if applicable) and attempts to abuse the system.

### 3. **Automated Tests** (optional)
- Write unit and integration tests for the following scenarios:
  - Deactivating an existing account.
  - Attempting to deactivate a non-existent account.
  - Attempting to deactivate by unauthorized users.
  - Deactivating an already deactivated account.

### 4. **Advanced Monitoring** (optional)
- Integrate monitoring tools such as **Prometheus** or **Grafana** to collect metrics on deactivation attempts and successes/failures.

### 5. **Scalability** (optional)
- Optimize the endpoint for handling large volumes of deactivations, potentially implementing a queuing system to ensure operations are handled efficiently.

---

## Next Steps
**Consider optional improvements** like implementing the reactivation endpoint, enhanced logging, and monitoring integration.
```
