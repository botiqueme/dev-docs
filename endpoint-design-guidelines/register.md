# **Endpoint: `/register`**

## **Purpose**
Register a new user, ensuring the data is valid, secure, and ready for use in a platform requiring email verification. This endpoint is designed to be secure, scalable, and unambiguous.

---

## **1. Technical Specifications**

### **Method**
`POST`

### **URL**
`/register`

### **Authentication**
None (public).

---

## **2. Parameters**

### **Required**
| **Name**       | **Type**  | **Description**                 | **Validation**                |
|----------------|-----------|-----------------------------------|--------------------------------|
| `email`        | String    | User's email.                  | Valid format, not disposable. |
| `password`     | String    | Password to be hashed.         | Minimum 8 characters, at least 1 uppercase, 1 number, 1 special character. |
| `name`         | String    | User's first name.             | Cannot be empty.              |
| `surname`      | String    | User's last name.              | Cannot be empty.              |

### **Optional**
| **Name**        | **Type**  | **Description**                 | **Validation**                |
|-----------------|-----------|---------------------------------|--------------------------------|
| `phone_number`  | String    | User's phone number.           | Regex for international formats. |
| `vat_number`    | String    | VAT number.                    | Regex for national formats.   |

### **System-Generated**
| **Name**       | **Type**  | **Description**                 | **Validation** |
|----------------|-----------|---------------------------------|----------------|
| `is_active`    | Boolean   | Indicates if the user account is active. | Default `True`. Cannot be modified at registration. |

---

### **3. Request Example**
```
POST /register  
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "name": "John",
  "surname": "Doe",
  "phone_number": "+1234567890",
  "vat_number": "IT12345678901"
}
```

---

## **4. Responses**

### **Success**
| **HTTP Status** | **Message**                          |
|-----------------|---------------------------------------|
| `201 Created`   | "User registered. Please verify your email." |

### **Errors**
| **Cause**                 | **HTTP Status**    | **Message**                                |
|----------------------------|--------------------|---------------------------------------------|
| Disposable email	          | `400 Bad Request` | "Disposable emails are not allowed."       |
| Duplicate email            | `409 Conflict`    | "Email already registered."                |
| Rate limit exceeded        | `429 Too Many Requests` | "Too many requests. Please try again later." |
| Database integrity issue   | `500 Error`       | "Internal Server Error, please contact the admin." |

---

## **5. Detailed Flow**

### **1. Request Handling**
1. Retrieve parameters sent by the frontend.
2. Ensure all required fields are present.

### **2. Validation**
- **Email**:
  - Valid format.
  - Not disposable (via `is_disposable_email` library).
- **Password**:
  - Minimum length of 8 characters.
  - Must contain at least 1 uppercase, 1 number, and 1 special character.
- **Optional Fields**:
  - Phone number (regex for international formats).
  - VAT number (regex for national formats).

### **3. User Creation**
1. Hash the password (`bcrypt`).
2. Generate a unique ID for the user (`UUID`).
3. Save the user to the database:
   - Ensure the email is not duplicated.
   - Automatically set `is_active = True`.

### **4. Email Verification**
1. Create a verification token using `URLSafeTimedSerializer`.
2. Send the token via an email containing a link to verify the account.

---

## **6. Implementation Code**

```
@v1.route('/register', methods=['POST'])
@limiter.limit("3 per minute")
def register():
    data = request.get_json()
    email = data['email']
    password = data['password']
    name = data['name']
    surname = data['surname']
    phone_number = data.get('phone_number')
    vat_number = data.get('vat_number')

    # Check if the email is disposable
    if is_disposable_email.check(email):
        return jsonify({"status": "error", "message": "Disposable emails are not allowed."}), 400

    # Hash the password
    password_hash = bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt())

    # Create the new user
    new_user = User(
        email=email,
        password_hash=password_hash,
        name=name,
        surname=surname,
        is_verified=False,
        is_active=True  # Automatically set active on registration
    )

    # Add optional fields
    if phone_number:
        new_user.phone_number = phone_number
    if vat_number:
        new_user.vat_number = vat_number

    # Save user to database
    try:
        db.session.add(new_user)
        db.session.commit()
    except IntegrityError:
        db.session.rollback()
        return jsonify({"status": "error", "message": "Email already registered."}), 409

    # Generate verification token and send email
    token = generate_verification_token(email)
    verify_url = url_for('v1.verify_email', token=token, _external=True)
    send_verification_email(email, verify_url)

    return jsonify({"status": "success", "message": "User registered. Please verify your email."}), 201
```

---

## **7. Database Integration**
- **`is_active` Field**:
  - Added to the `users` table as a boolean.
  - Default value: `True` (active users).
  - Used to manage account status:
    - If a user deactivates their account, set `is_active = False`.
    - If the account is reactivated, set `is_active = True`.

#### **Database Migration**
```
ALTER TABLE users ADD COLUMN is_active BOOLEAN DEFAULT TRUE;
```

---

## **8. Security Details**
1. **Rate Limiting**:
   - 3 requests per minute per IP using Flask-Limiter.
2. **Secure Passwords**:
   - Hashed with bcrypt.
3. **Blocked Disposable Emails**:
   - Validated through a dedicated library.
4. **Email Verification Token**:
   - Securely generated using `URLSafeTimedSerializer`.

---

## **9. Future Considerations**
- **Inactive Users**:
  - Ensure `/login` endpoint checks `is_active` status.
  - Add `/reactivate_user` to handle account reactivation.
- **Monitoring**:
  - Track registration and verification metrics with tools like Prometheus.
- **Scalability**:
  - Implement email sending via a queue to handle spikes in user registration.
