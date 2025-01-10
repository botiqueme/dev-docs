# Endpoint Documentation: `/verify_email`

## Purpose
Validate a user's email through a unique token generated during registration.

## Technical Specifications

### Method
- **GET**

### URL
- `/verify_email`
- 
### Authentication
- None (public).

### Parameters

| Parameter | Type   | Required | Description                              |
|-----------|--------|----------|------------------------------------------|
| token     | String | Yes      | Unique token to verify the email address.|

---

## Endpoint Logic
1. **Token Validation**:
   - Check if the token is present in the request.
   - Verify the token is not blacklisted.
   - Decode the token and ensure it is valid (not expired or corrupted).
2. **User Retrieval**:
   - Use the email decoded from the token to find the user in the database.
   - Return an error if the user is not found.
3. **User Status Verification**:
   - Check if the user is already verified.
   - If yes, return an error indicating the user is already verified.
4. **Status Update**:
   - Update the user's `is_verified` field to `True`.
5. **Logging**:
   - Log successful and failed verification attempts for monitoring.

## Responses

### Success

| HTTP Status | Message                        |
|-------------|--------------------------------|
| 200 OK      | "Email verified successfully." |


### Errors

| Cause                  | HTTP Status       | Message                                     |
|------------------------|-------------------|---------------------------------------------|
| Missing Token          | 400 Bad Request  | "Missing token."                            |
| Invalid Token          | 401 Unauthorized | "Invalid token."                            |
| Expired Token          | 401 Unauthorized | "Expired token."                            |
| User Not Found         | 404 Not Found    | "User not found."                           |
| User Already Verified  | 409 Conflict     | "User is already verified."                 |
| Rate Limit Exceeded    | 429 Too Many Requests | "Too many attempts. Try again later."    |

---

## Codice di Implementazione

```
@v1.route('/verify_email/<token>', methods=['GET'])
@limiter.limit("5 per minute", override_defaults=False, deduct_when=lambda resp: resp.status_code == 200)
@limiter.limit("3 per minute", override_defaults=False, deduct_when=lambda resp: resp.status_code != 200)
def verify_email(token):
    # Controllo presenza del token
    if not token:
        return jsonify({"Bad Request": "Missing token."}), 400
    
    # Controlla se il token è nella blacklist
    if BlacklistToken.query.filter_by(token=token).first():
        # Verifica il tipo di risposta preferito dal client
        if request.accept_mimetypes.accept_json and not request.accept_mimetypes.accept_html:
            return jsonify({"Unauthorized": "Blacklisted token."}), 401
        else:
            return redirect(url_for('v1.invalid_token'))
    
    try:
        serializer = URLSafeTimedSerializer(current_app.config['SECRET_KEY'])
        email = serializer.loads(
            token,
            salt=current_app.config['SECURITY_PASSWORD_SALT'],
            max_age=86400  # 24 ore
        )
    except SignatureExpired:
        # Token scaduto
        if request.accept_mimetypes.accept_json and not request.accept_mimetypes.accept_html:
            return jsonify({"Unauthorized": "Expired token."}), 401
        else:
            return redirect(url_for('v1.expired_token', email=email))
    except BadSignature:
        # Token non valido
        if request.accept_mimetypes.accept_json and not request.accept_mimetypes.accept_html:
            return jsonify({"Unauthorized": "Invalid token."}), 401
        else:
            return redirect(url_for('v1.invalid_token'))

    # Verifica utente nel database
    user = User.query.filter_by(email=email).first()
    if user is None:
        if request.accept_mimetypes.accept_json and not request.accept_mimetypes.accept_html:
            return jsonify({"Not Found": "User not found."}), 404
        else:
            return redirect(url_for('v1.user_not_found'))
    if user.is_verified:
        if request.accept_mimetypes.accept_json and not request.accept_mimetypes.accept_html:
            return jsonify({"Conflict": "User is already verified."}), 409
        else:
            return redirect(url_for('v1.verified_token'))

    # Aggiorna stato di verifica
    user.is_verified = True
    db.session.commit()

    # Aggiungi il token alla blacklist
    new_blacklist_entry = BlacklistToken(token=token)
    db.session.add(new_blacklist_entry)
    db.session.commit()

    if request.accept_mimetypes.accept_json and not request.accept_mimetypes.accept_html:
        return jsonify({"OK": "Email verified successfully."}), 200
    else:
        return redirect(url_for('v1.verified_token'))
```

## Next Steps for the `/verify_email` Endpoint

### 1. Comprehensive Monitoring (optional improvement)

**Objective**  
Integrate tools like Prometheus or Grafana to collect and analyze detailed metrics on email verifications.

**Necessary Actions**  
- **Configure Prometheus Flask Exporter**:
    - Collect metrics such as:
        - Number of successful verifications.
        - Failed attempts (e.g., invalid or expired tokens).
        - Frequency of invalidated tokens.
- **Integrate Grafana**:
    - Create dashboards to visualize the metrics collected from Prometheus.
    - Add charts to monitor email verification trends.

### 2. Frontend Page (optional improvement)

**Objective**  
Enhance user experience by providing a dedicated page with visual feedback based on the verification status.

**Necessary Actions**  
- Create an HTML page for the following states:
    - **Success**: "Your email has been successfully verified."
    - **Expired Token**: "The link has expired. Request a new token."
        - Button: "Request New Token" (linked to `/resend_verification`).
    - **Invalid Token**: "Error during verification. Please contact support."

---
### 3. Automated Testing (optional improvement)


markdown
Copia codice
# Endpoint Documentation: `/verify_email`

---

## Purpose
Validate a user's email through a unique token generated during registration.

---

## Technical Specifications

### Method
- **GET**

### URL
- `/verify_email`

### Authentication
- None (public).

### Parameters

| Parameter | Type   | Required | Description                              |
|-----------|--------|----------|------------------------------------------|
| token     | String | Yes      | Unique token to verify the email address.|

---

## Endpoint Logic
1. **Token Validation**:
   - Check if the token is present in the request.
   - Verify the token is not blacklisted.
   - Decode the token and ensure it is valid (not expired or corrupted).
2. **User Retrieval**:
   - Use the email decoded from the token to find the user in the database.
   - Return an error if the user is not found.
3. **User Status Verification**:
   - Check if the user is already verified.
   - If yes, return an error indicating the user is already verified.
4. **Status Update**:
   - Update the user's `is_verified` field to `True`.
5. **Logging**:
   - Log successful and failed verification attempts for monitoring.

---

## Responses

### Success

| HTTP Status | Message                        |
|-------------|--------------------------------|
| 200 OK      | "Email verified successfully." |

---

### Errors

| Cause                  | HTTP Status       | Message                                     |
|------------------------|-------------------|---------------------------------------------|
| Missing Token          | 400 Bad Request  | "Missing token."                            |
| Invalid Token          | 401 Unauthorized | "Invalid token."                            |
| Expired Token          | 401 Unauthorized | "Expired token."                            |
| User Not Found         | 404 Not Found    | "User not found."                           |
| User Already Verified  | 409 Conflict     | "User is already verified."                 |
| Rate Limit Exceeded    | 429 Too Many Requests | "Too many attempts. Try again later."    |

---

## Next Steps for the `/verify_email` Endpoint

### 1. Comprehensive Monitoring (optional improvement)

**Objective**  
Integrate tools like Prometheus or Grafana to collect and analyze detailed metrics on email verifications.

**Necessary Actions**  
- **Configure Prometheus Flask Exporter**:
    - Collect metrics such as:
        - Number of successful verifications.
        - Failed attempts (e.g., invalid or expired tokens).
        - Frequency of invalidated tokens.
- **Integrate Grafana**:
    - Create dashboards to visualize the metrics collected from Prometheus.
    - Add charts to monitor email verification trends.

---

### 2. Frontend Page (optional improvement)

**Objective**  
Enhance user experience by providing a dedicated page with visual feedback based on the verification status.

**Necessary Actions**  
- Create an HTML page for the following states:
    - **Success**: "Your email has been successfully verified."
    - **Expired Token**: "The link has expired. Request a new token."
        - Button: "Request New Token" (linked to `/resend_verification`).
    - **Invalid Token**: "Error during verification. Please contact support."

---

### 3. Automated Testing (optional improvement)

**Objective**  
Ensure the quality and correct functioning of the endpoint in all use cases.

**Necessary Actions**  
- Create unit and integration tests to cover:
    - Valid Tokens:
        - Test that a valid token successfully verifies the user.
    - Invalid Tokens:
        - Test for expired or invalid signatures.
    - Blacklist:
        - Ensure blacklisted tokens are rejected.
    - Rate Limiting:
        - Validate correct application of rate limits (e.g., 5 requests per minute for valid tokens, 3 for invalid tokens).

- **Esempio di test**:
    ```
    def test_verify_email_success(client):
        response = client.get('/verify_email?token=valid_token')
        assert response.status_code == 200
        assert response.json['message'] == "Email verified successfully."

    def test_verify_email_blacklisted(client):
        response = client.get('/verify_email?token=blacklisted_token')
        assert response.status_code == 401
        assert response.json['message'] == "Token invalidated."
    ```

### 4. Scalability Optimization (optional improvement)

**Objective**  
Ensure the system remains performant in environments with high concurrent requests.

**Necessary Actions**  
- **Distributed Blacklist**:
    - Use a distributed system like Redis to synchronize the blacklist across multiple backend instances.
- **Load Testing**:
    - Perform load testing with tools like Apache JMeter or k6.
    - Monitor resource usage during high-intensity testing.


## Priorities
1. **Frontend Page**: Immediately improve user experience.
2. **Automated Testing**: Ensure system quality and stability.
3. **Comprehensive Monitoring**: Enable continuous metric analysis.
4. **Scalability Optimization**: Essential for high-traffic environments.

