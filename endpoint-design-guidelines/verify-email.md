# **Endpoint: `/verify_email`**

## **Purpose**
Validates a user's email using a unique token generated at registration.

---

## **1. Technical Specifications**

### **Method**
- **GET**

### **URL**
- `/verify_email`

### **Authentication**
- None (public).

### **Parameters**
| **Parameter** | **Type** | **Required** | **Description** |
|--------------|---------|------------|--------------------------------------------------|
| `token` | String | Yes | Unique token used to verify the email address. |

---

## **2. Endpoint Logic**
1. **Token Validation**:
   - Check if the token is present in the request.
   - Verify the token is not blacklisted.
   - Decode and validate the token (ensure it's not expired or tampered with).

2. **User Retrieval**:
   - Extract the email from the decoded token.
   - Search for the user in the database using the extracted email.
   - Return an error if the user does not exist.

3. **User Status Check (`is_verified` and `is_active`)**:
   - If the user is already verified, return an error.
   - **If `is_active = False`, re-enable the account (`is_active = True`).**

4. **Status Update**:
   - Update the `is_verified` field to `True`.
   - If the user was previously deactivated (`is_active = False`), set `is_active = True`.

5. **Logging & Token Blacklisting**:
   - Log all verification attempts (successful and failed).
   - Add the used token to the blacklist to prevent reuse.

---

## **3. Responses**

### ✅ **Success**
| **HTTP Status** | **Message** |
|---------------|------------------------------|
| `200 OK` | "Email verified successfully." |

**Example Response:**
```json
{
  "status": "success",
  "code": 200,
  "message": "Email verified successfully."
}
```

---

### ❌ **Errors**
#### **1️⃣ Missing Token**
| **HTTP Status** | **Message** |
|---------------|------------------------------|
| `400 Bad Request` | "Missing token." |

```json
{
  "status": "error",
  "code": 400,
  "message": "Missing token."
}
```

---

#### **2️⃣ Invalid Token**
| **HTTP Status** | **Message** |
|---------------|------------------------------|
| `401 Unauthorized` | "Invalid token." |

```json
{
  "status": "error",
  "code": 401,
  "message": "Invalid token."
}
```

---

#### **3️⃣ Expired Token**
| **HTTP Status** | **Message** |
|---------------|------------------------------|
| `401 Unauthorized` | "Expired token." |

```json
{
  "status": "error",
  "code": 401,
  "message": "Expired token."
}
```

---

#### **4️⃣ User Not Found**
| **HTTP Status** | **Message** |
|---------------|------------------------------|
| `404 Not Found` | "User not found." |

```json
{
  "status": "error",
  "code": 404,
  "message": "User not found."
}
```

---

#### **5️⃣ User Already Verified**
| **HTTP Status** | **Message** |
|---------------|------------------------------|
| `409 Conflict` | "User is already verified." |

```json
{
  "status": "error",
  "code": 409,
  "message": "User is already verified."
}
```

---

## **4. Updated Code Implementation**

```python
@v1.route('/verify_email/<token>', methods=['GET'])
@limiter.limit("5 per minute", override_defaults=False, deduct_when=lambda resp: resp.status_code == 200)
@limiter.limit("3 per minute", override_defaults=False, deduct_when=lambda resp: resp.status_code != 200)
def verify_email(token):
    # Check if token is provided
    if not token:
        return jsonify({"error": "Missing token."}), 400

    # Check if token is blacklisted
    if BlacklistToken.query.filter_by(token=token).first():
        return jsonify({"error": "Blacklisted token."}), 401

    try:
        # Decode the token
        serializer = URLSafeTimedSerializer(current_app.config['SECRET_KEY'])
        email = serializer.loads(
            token,
            salt=current_app.config['SECURITY_PASSWORD_SALT'],
            max_age=86400  # 24 hours validity
        )
    except SignatureExpired:
        return jsonify({"error": "Expired token."}), 401
    except BadSignature:
        return jsonify({"error": "Invalid token."}), 401

    # Retrieve user from database
    user = User.query.filter_by(email=email).first()
    if user is None:
        return jsonify({"error": "User not found."}), 404

    # Check if the user is already verified
    if user.is_verified:
        return jsonify({"error": "User is already verified."}), 409

    # Update user verification status
    user.is_verified = True

    # Reactivate the user if previously deactivated
    if not user.is_active:
        user.is_active = True

    db.session.commit()

    # Blacklist the used token
    new_blacklist_entry = BlacklistToken(token=token)
    db.session.add(new_blacklist_entry)
    db.session.commit()

    return jsonify({"success": "Email verified successfully."}), 200
```

---

## **5. Next Steps for the `/verify_email` Endpoint**

### **1️⃣ Reactivation Upon Verification**
✅ **Added logic to re-enable inactive accounts (`is_active = True`).**  
✅ **Ensures that users who verify their email regain access to the platform.**

---

### **2️⃣ Comprehensive Monitoring (optional improvement)**

#### **Objective**  
Integrate Prometheus or Grafana to track verification trends.

#### **Actions**
- **Collect Metrics**:
  - Number of successful verifications.
  - Failed attempts due to invalid/expired tokens.
  - Frequency of blacklisted tokens.
- **Visualization**:
  - Set up dashboards to display verification patterns.

---

### **3️⃣ Frontend Page for Better UX (optional improvement)**

#### **Objective**  
Improve user experience with a verification status page.

#### **Actions**
- **Create an HTML page**:
  - **Success**: "Your email has been successfully verified."
  - **Expired Token**: "The link has expired. Request a new one."
  - **Invalid Token**: "Error during verification. Contact support."

---

### **4️⃣ Automated Testing (optional improvement)**

#### **Objective**  
Ensure correctness across various verification scenarios.

#### **Actions**
- **Test cases**:
  - ✅ Successful verification with a valid token.
  - ❌ Attempting to verify with an expired/invalid token.
  - ❌ Blacklisted token rejection.
  - ❌ Rate limit enforcement.

- **Example Unit Test**:
```python
def test_verify_email_success(client):
    response = client.get('/verify_email?token=valid_token')
    assert response.status_code == 200
    assert response.json['message'] == "Email verified successfully."

def test_verify_email_blacklisted(client):
    response = client.get('/verify_email?token=blacklisted_token')
    assert response.status_code == 401
    assert response.json['message'] == "Blacklisted token."
```

---

### **5️⃣ Scalability & Security Enhancements (optional improvement)**

#### **Objective**  
Ensure the endpoint remains performant at scale.

#### **Actions**
- **Distributed Blacklist Storage**:
  - Use **Redis** to sync blacklisted tokens across instances.
- **Load Testing**:
  - Simulate thousands of requests using **JMeter** or **k6**.
- **Rate Limit Adjustments**:
  - Fine-tune request limits based on traffic patterns.
