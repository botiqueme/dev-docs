# **Endpoint: `/resend_verification`**

## **Purpose**
Allows users to request a new email verification token if the previous one has expired or is invalid. This endpoint ensures a smooth, secure, and compliant user experience.

---

## **1. Technical Details**

### **Method**
`POST`

### **URL**
`/resend_verification`

### **Authentication**
None (public).

---

## **2. Request Parameters**

| **Parameter** | **Type**  | **Required** | **Description**                           |
|--------------|-----------|-------------|-------------------------------------------|
| `email`      | String    | Yes         | The email address associated with the user account. |

#### **Example Request**
```
{
  "email": "user@example.com"
}
```

---

## **3. Endpoint Logic**

### **Key Steps**
1. **Input Validation**:
   - Check that the `email` parameter is provided.

2. **User Retrieval**:
   - Search the database for a user associated with the given email.
   - **Handle errors**:
     - User not found.
     - User already verified.
     - **User deactivated (`is_active = False`)**.

3. **User Status Check**:
   - If the user is **already verified**, return an appropriate response.
   - **If the user is deactivated (`is_active = False`)**, block the request and return `403 Forbidden`.

4. **Token Generation**:
   - Use `URLSafeTimedSerializer` to create a new unique token.
   - Configure token expiration (e.g., 24 hours).

5. **Email Delivery**:
   - Send the verification email containing the token and verification link.

6. **Rate Limiting**:
   - Limit requests to **3 per hour per IP/email**.
   - Implement detailed logging for suspicious attempts.

7. **Logging & Monitoring (optional improvement)**:
   - Log every token regeneration attempt (success/failure).
   - Collect metrics for future analysis.

---

## **4. Endpoint Responses**

| **Scenario**               | **HTTP Status**   | **Message**                                |
|----------------------------|-------------------|--------------------------------------------|
| **Success**                | `200 OK`         | "Verification email resent."              |
| **Missing Email**          | `400 Bad Request` | "Missing email."                          |
| **User Not Found**         | `404 Not Found`   | "User not found."                         |
| **User Already Verified**  | `409 Conflict`    | "User is already verified."               |
| **User Deactivated**       | `403 Forbidden`   | "Account is inactive. Please contact support." |
| **Rate Limit Exceeded**    | `429 Too Many Requests` | "Too many requests. Please try again later." |
| **Internal Error**         | `500 Internal Server Error` | "Internal (Re-send verification) Server Error, please contact the admin." |

#### **Example Success Response**
```
{
  "status": "success",
  "code": 200,
  "message": "Verification email resent."
}
```

---

## **5. Implementation Code**

```
@v1.route('/resend_verification', methods=['POST'])
@limiter.limit("3 per hour")  # Limit to 3 requests per hour per user
def resend_verification():
    """
    Resends a verification email to a user.

    Returns:
        Response: A response with a message and a status code.
    """
    # Validate input
    email = request.form.get('email')
    if not email:
        return jsonify({"status": "error", "code": 400, "message": "Missing email."}), 400

    # Check if email exists in the database
    user = User.query.filter_by(email=email).first()
    if not user:
        return jsonify({"status": "error", "code": 404, "message": "User not found"}), 404
    
    # Check if the user is already verified
    if user.is_verified:
        return jsonify({"status": "error", "code": 409, "message": "User is already verified."}), 409
    
    # Block request if user is deactivated
    if not user.is_active:
        return jsonify({"status": "error", "code": 403, "message": "Account is inactive. Please contact support."}), 403

    try:
        # Generate a new verification token
        token = utils.generate_verification_token(email)
        verify_url = url_for('v1.verify_email', token=token, email=email, _external=True)

        response = utils.send_verification_email(email, verify_url)
        if response.status_code == 404:
            callback_refresh()
            response = utils.send_verification_email(email, verify_url)
    except Exception as e:
        current_app.logger.info(e)
        return jsonify({"status": "error", "code": 500, "message": "Internal (Re-send verification) Server Error, please contact the admin."}), 500

    return jsonify({"status": "success", "code": 200, "message": "Verification email resent."}), 200
```

---

## **6. Security Considerations & Implemented Improvements**

1. **Rate Limiting**:
   - Configured to prevent abuse and brute-force attacks.

2. **Input Validation**:
   - Strict email format validation.

3. **Handling Deactivated Accounts (`is_active`)**:
   - **Prevents deactivated users from requesting verification tokens** (`403 Forbidden`).

4. **Error Handling**:
   - Detailed error handling for improved user experience and security.

5. **Logging (optional improvement)**:
   - Advanced monitoring of success and failure rates to prevent abuse.

---

## **7. Next Steps**

1. **Automated Testing**:
   - Test cases:
     - Missing email.
     - User not found.
     - User already verified.
     - User deactivated (`is_active = False`).
     - Rate limit enforcement.

2. **Advanced Monitoring (optional improvement)**:
   - Integrate **Prometheus** to collect metrics:
     - Number of email resends.
     - Frequency of errors.

3. **Frontend Integration**:
   - Create a UI to notify users about request status:
     - "Verification link sent."
     - "Error during token regeneration."
     - "Account deactivated, please contact support."
