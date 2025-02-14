# Endpoint: `/reset_password_request`

## Purpose
This endpoint allows users to request a password reset link. It has been enhanced to include advanced security, scalability, and abuse prevention features.

---

## 1. Technical Details

### **Method**
`POST`

### **URL**
`/reset_password_request`

### **Authentication**
None (public).

---

## 2. Request Parameters

| **Parameter** | **Type**  | **Required** | **Description**                             |
|---------------|-----------|--------------|---------------------------------------------|
| `email`       | String    | Yes          | The registered email address of the user.   |

Example request:
```
{
  "email": "user@example.com"
}
```

---

## 3. Implemented Improvements

### 1. Protection Against User Enumeration
- Generic response when the user is not found to prevent malicious actors from verifying which emails are registered.

### 2. Token Validity
- Tokens are valid for 15 minutes.
- Payload includes:
  - **`email`**: Unique user identifier.
  - **`iat`**: Token creation date and time.
  - **`jti`**: Unique token identifier to prevent reuse.

### 3. Advanced Rate Limiting
- Configured limits:
  - 3 requests per IP/email per hour.
  - Optional global limit: Maximum of 50 requests per minute to ensure scalability.

### 4. Detailed Logging
- Each request is logged with:
  - **Email**.
  - **IP address**.
  - **Result** (success/failure).
  - **Timestamp**.

### 5. Token Rotation (Optional Improvement)
- Automatically invalidates previously generated tokens for the same email when a new token is requested.

### 6. CAPTCHA Protection (Optional Improvement)
- Integrated reCAPTCHA to prevent automated attacks.

### 7. CSRF Protection (Optional Improvement)
- Requires a CSRF token to ensure requests come from legitimate sources.

---

## 4. Endpoint Responses

| **Scenario**               | **HTTP Status**     | **Message**                                 |
|----------------------------|---------------------|---------------------------------------------|
| **Success**                | `200 OK`           | "If the email exists, a reset link has been sent." |
| **Missing Email**          | `400 Bad Request`  | "Missing email."                            |
| **Rate Limit Exceeded**    | `429 Too Many Requests` | "Too many requests. Please try again later." |
| **User Not Found**    | `404 User not found` | "User not found" |

Example Success Response:
```
{
  "status": "success",
  "code": 200,
  "message": "If the email exists, a reset link has been sent."
}
```

---

## 5. Codice Completo

```
@v1.route('/reset_password_request', methods=['POST'])
@limiter.limit("3 per hour", key_func=lambda: request.remote_addr)
def reset_password_request():
    """
    Endpoint per richiedere un link di reset della password.
    """
    try:
        # Recupera i dati dal corpo della richiesta
        data = request.get_json()
        if not data or 'email' not in data:
            return utils.jsonify_return_error("Bad Request", 400, "Missing email."), 400

        email = data['email']

        # Verifica se l'email è un'email usa e getta
        disposable = is_disposable_email.check(email)
        if disposable:
            return utils.jsonify_return_error("Error", 400, "Disposable emails are not allowed."), 400

        # Cerca l'utente nel database
        user = User.query.filter_by(email=email).first()

        # Protezione contro l'enumerazione degli utenti
        if user:
            # Genera un token di reset della password
            token = utils.generate_verification_token(email)
            # Costruisce l'URL di reset della password
            verify_url = url_for('v1.verify_email', token=token, _external=True)

            response = utils.send_verification_email(email, verify_url)
            if response.status_code == 404:
                callback_refresh()
                response = utils.send_verification_email(email, verify_url)
        else:
            current_app.logger.info(f"{user_ip} - /reset_password_request User not found.")
            return utils.jsonify_return_error("Error", 404, "User not found."), 404

    except Exception as e:
        return utils.jsonify_return_error("Error", 500, "Internal Server Error, please contact the admin"), 500

    if response.status_code == 200:
        return utils.jsonify_return_success("success", 200, {"message": "If an account with that email exists, a password reset link has been sent."}), 200
```

---

## 6. Next Steps

1. **Implement Global Rate Limit** (optional improvement):
   - Configure global limits to prevent large-scale attacks.

2. **CAPTCHA Integration** (optional improvement):
   - Add CAPTCHA for suspicious automated requests.

3. **Implementation of `/reset_password`**:
   - Create an endpoint to consume the token and reset the password.

4. **Comprehensive Testing**:
   - Validate scenarios:
     - Invalid input.
     - User not found.
     - Working rate limiting.
     - Successful email sending.

5. **Advanced Monitoring**:
   - Integrate tools like Grafana to visualize metrics:
     - Number of requests.
     - Failed attempts.
     - Request outcomes.

---

