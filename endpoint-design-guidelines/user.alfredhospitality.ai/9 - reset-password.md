# Endpoint: `/reset_password`

## Purpose
Allows users to securely reset their password using a previously generated reset token. This updated version implements significant improvements in security and scalability.

---

## 1. Technical Details

### **Method**
`POST`

### **URL**
`/reset_password`

### **Authentication**
None (public).

---

## 2. Request Parameters

| **Parameter**  | **Type**   | **Required** | **Description**                                      |
|----------------|------------|--------------|------------------------------------------------------|
| `token`        | String     | Yes          | Reset token received via email.                     |
| `new_password` | String     | Yes          | The new password the user wants to set.             |

Example request:
```
{
  "token": "example_reset_token",
  "new_password": "NewPassword123!"
}
```

---

## 3. Endpoint Logic Flow

### **1. Input Validation**
- **Token**:
  - Verify it is present and correctly formatted.
- **Password**:
  - Must meet security criteria:
    - Minimum length of 8 characters.
    - Must include:
      - An uppercase letter.
      - A lowercase letter.
      - A number.
      - A special character.

### **2. Token Decoding and Validation**
- Use `URLSafeTimedSerializer` to:
  - Decode the token.
  - Verify it is not expired (valid for 15 minutes).
  - Extract the associated email.
- **Error Handling**:
  - **Expired Token**: Return a specific error.
  - **Invalid Token**: Return a specific error.

### **3. Protection Against Race Conditions (optional improvement)**
- Use a temporary lock to prevent simultaneous use of the same token in concurrent requests.

### **4. User Retrieval**
- Search for the user associated with the email extracted from the token.
- Return an error if the user does not exist.

### **5. Password Update**
- Hash the new password using `bcrypt`.
- Update the `password_hash` field in the database.

### **6. Token Invalidation**
- Add a unique identifier (`jti`) to the token.
- Store used tokens in a blacklist to invalidate them after use.

### **7. Detailed Logging**
- Log:
  - Emails associated with the request.
  - Outcomes (success/failure).
  - Timestamp.

### **8. Global Rate Limiting (optional improvement)**
- Limit consecutive requests by IP or email to prevent attacks.

---

## 4. Endpoint Responses

| **Scenario**               | **HTTP Status**     | **Message**                                |
|----------------------------|---------------------|---------------------------------------------|
| **Success**                | `200 OK`           | "Password reset successfully."              |
| **Missing/Invalid Token**  | `400 Bad Request`  | "Invalid or expired token."                 |
| **Invalid Password**       | `400 Bad Request`  | "Invalid password format."                  |
| **User Not Found**         | `404 Not Found`    | "User not found."                           |


Esempio di Risposta Successo:
```
{
  "status": "success",
  "code": 200,
  "message": "Password reset successfully."
}
```

---

## 5. Codice Aggiornato

```
from flask import request, jsonify, current_app
from itsdangerous import URLSafeTimedSerializer, SignatureExpired, BadSignature
from app.models import User
from app import db, blacklist
import bcrypt
import logging
from threading import Lock

# Configurazione logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Lock per protezione contro race condition
reset_lock = Lock()

@v1.route('/reset_password', methods=['POST'])
def reset_password():
    data = request.get_json()
    token = data.get('token')
    new_password = data.get('new_password')

    # Validazione input
    if not token or not new_password:
        logger.warning("Reset password failed: Missing token or new_password.")
        return jsonify({
            "status": "error",
            "code": 400,
            "message": "Missing token or new_password."
        }), 400

    # Validazione password
    if len(new_password) < 8 or not any(char.isdigit() for char in new_password) or not any(char.isalpha() for char in new_password):
        logger.warning("Reset password failed: Invalid password format.")
        return jsonify({
            "status": "error",
            "code": 400,
            "message": "Invalid password format."
        }), 400

    # Decodifica del token
    try:
        serializer = URLSafeTimedSerializer(current_app.config['SECRET_KEY'])
        email = serializer.loads(
            token,
            salt=current_app.config['SECURITY_PASSWORD_SALT'],
            max_age=900  # Token valido per 15 minuti
        )
    except SignatureExpired:
        logger.warning("Reset password failed: Token expired.")
        return jsonify({
            "status": "error",
            "code": 400,
            "message": "Invalid or expired token."
        }), 400
    except BadSignature:
        logger.warning("Reset password failed: Invalid token.")
        return jsonify({
            "status": "error",
            "code": 400,
            "message": "Invalid or expired token."
        }), 400

    # Protezione contro race condition
    with reset_lock:
        if token in blacklist:
            logger.warning("Reset password failed: Token already used.")
            return jsonify({
                "status": "error",
                "code": 400,
                "message": "Invalid or expired token."
            }), 400

        # Recupero utente
        user = User.query.filter_by(email=email).first()
        if not user:
            logger.warning(f"Reset password failed: User not found for email {email}.")
            return jsonify({
                "status": "error",
                "code": 404,
                "message": "User not found."
            }), 404

        # Aggiornamento password
        try:
            hashed_password = bcrypt.hashpw(new_password.encode('utf-8'), bcrypt.gensalt())
            user.password_hash = hashed_password
            db.session.commit()
            logger.info(f"Password reset successfully for user {email}.")
        except Exception as e:
            logger.error(f"Error updating password: {e}")
            return jsonify({
                "status": "error",
                "code": 500,
                "message": "Error resetting password."
            }), 500

        # Invalidazione del token
        blacklist.add(token)

    return jsonify({
        "status": "success",
        "code": 200,
        "message": "Password reset successfully."
    }), 200
```

---

## 6. Security and Next Steps

### **Security**
1. **Token Rotation**:
   - Implemented with the use of a centralized blacklist.
2. **CSRF Protection (optional improvement)**:
   - Add a CSRF token to ensure requests originate from legitimate sources.

### **Next Steps**
1. **Comprehensive Testing**:
   - Validate complex scenarios:
     - Expired or invalid tokens.
     - Non-compliant passwords.
     - Non-existent users.
2. **Frontend Integration**:
   - Create a dedicated UI for entering the token and new password.
3. **Monitoring and Analysis**:
   - Use monitoring tools (e.g., Grafana) to collect statistics on endpoint usage.

---

