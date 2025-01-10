# Endpoint: `/register`

## Purpose
Register a new user, ensuring the data is valid, secure, and ready for use in a platform requiring email verification. This endpoint is designed to be secure, scalable, and unambiguous.

---

## Technical Specifications

### **Method**
`POST`

### **URL**
`/register`

### **Authentication**
None (public).

---

## Parameters

### **Required**
| **Name**       | **Type**  | **Description**                 | **Validation**                |
|---------------|-----------|-----------------------------------|---------------------------------|
| `email`        | String    | User's email.                  | Valid format, not disposable. |
| `password`     | String    | Password to be hashed.         | Minimum 8 characters, at least 1 uppercase, 1 number, 1 special character. |
| `name`         | String    | User's first name.             | Cannot be empty.              |
| `surname`      | String    | User's last name.              | Cannot be empty.              |

### **Optional**
| **Name**        | **Type**  | **Description**                 | **Validation**                |
|-----------------|-----------|---------------------------------|--------------------------------|
| `phone_number`  | String    | User's phone number.           | Regex for international formats. |
| `vat_number`    | String    | VAT number.                    | Regex for national formats.   |

### Request Example
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

## Responses

### **Success**
| **HTTP Status** | **Message**                          |
|-----------------|---------------------------------------|
| `201 Created`   | "User registered. Please verify your email." |

### **Errors**
| **Causs**                  | **HTTP Status** | **Message**                                |
|----------------------------|-----------------|---------------------------------------------|
| Disposable email	          | `400 Bad Request` | "Disposable emails are not allowed."       |
| Duplicate email            | `409 Conflict`    | "Email already registered."                |
| Rate limit exceeded        | `429 Too Many Requests` | "Too many requests. Please try again later." |
| Database integrity issue   | `500 Error` | "Internal Server Error, please contact the admin" |

---

## Detailed Flow

### **1. Request Handling**
1. Retrieve parameters sent by the frontend.
2. Ensure all required fields are present (handled by frontend).

### **2. Validation**
- **Email**:
  - Valid format (handled by frontend).
  - Not disposable (via `is_disposable_email` library).
- **Password**:
  - Minimum length of 8 characters (handled by frontend).
  - Must contain at least 1 uppercase, 1 number, and 1 special character (handled by frontend).
- **Optional Fields**:
  - Phone number (regex for international formats - handled by frontend).
  - VAT number (regex for national formats - handled by frontend).

### **3. User Creation**
1. Hash the password (`bcrypt`).
2. Generate a unique ID for the user (`UUID`).
3. Save the user to the database:
   - Ensure the email is not duplicated.

### **4. Email Verification**
1. Create a verification token using `URLSafeTimedSerializer`.
2. Send the token via an email containing a link to verify the account.

### **5. Responses**
- Success: Confirm the user has been registered and request email verification.
- Error: Return an appropriate message for each encountered issue.

---

## Implementation Code

```
@v1.route('/register', methods=['POST'])
@limiter.limit("3 per minute")
def register():
    current_app.logger.info("Registering new user...")

    data = request.get_json()
    email = data['email']
    password = data['password']
    name = data['name']
    surname = data['surname']
    phone_number = data.get('phone_number')
    vat_number = data.get('vat_number')

    # Verifica se l'email è un'email usa e getta
    disposable = is_disposable_email.check(email)
    if disposable:
        return jsonify_return_error("Error", 400, "Disposable emails are not allowed."), 400
    

    # Hash della password
    password_hash = utils.hash_password(password)


    # Creazione nuovo utente
    new_user = User(
        email=email,
        password_hash=password_hash,
        name=name,
        surname=surname,
        is_verified=False
    )

    # Aggiunta dei campi opzionali solo se non sono vuoti
    if phone_number:
        new_user.phone_number = phone_number
    if vat_number:
        new_user.vat_number = vat_number
    
    current_app.logger.info("Checking if email is already registered...")

    # Verifica se l'email è già registrata
    existing_user = User.query.filter_by(email=email).first()
    if existing_user:
        return jsonify_return_error("Conflict", 409, "Email already registered"), 409

    # Aggiungi l'utente al database
    try:
        db.session.add(new_user)
        db.session.commit()
    except IntegrityError as e:
        db.session.rollback()
        return jsonify_return_error("Error", 500, "Internal (Integrity) Server Error, please contact the admin"), 500
    except Exception as e:
        db.session.rollback()
        return jsonify_return_error("Error", 500, "Internal (Databse) Server Error, please contact the admin"), 500

    try:
        # Generazione token di verifica
        token = utils.generate_verification_token(email)
        # logger.info(email)
        verify_url = url_for('v1.verify_email', token=token, email=email, _external=True)
        # logger.info(verify_url)

        response = utils.send_verification_email(email, verify_url)
        # logger.info(response)
        if response.status_code == 404:
            callback_refresh()
            response = utils.send_verification_email(email, verify_url)
    except Exception as e:
        current_app.logger.info(e)
        return jsonify_return_error("Error", 500, "Internal (Verification Email) Server Error, please contact the admin"), 500


    if response.status_code == 200:
        return jsonify_return_success("success", 201, {"message": "User registered. Please verify your email."}), 201
    else:
        return jsonify_return_error("Error", 500, "Internal (Generic) Server Error, please contact the admin"), 500

```

---

## Security Details
1. **Rate Limiting**:
   - 3 requests per minute per IP using Flask-Limiter.
2. **Secure Passwords**:
   - Hashed with bcrypt.
3. **Blocked Disposable Emails:**:
   - Validation through a dedicated library.
4. **Secure Email Verification Token:**:
   - Using URLSafeTimedSerializer.

---

## Future Considerations and Suggested Improvements

### 1. Handling Unverified Users
- **Problem**: Users who do not verify their email may remain in the database indefinitely.
- **Solution**:
  - Implement a system to automatically delete unverified users after a period (e.g., 7 days).

### 2. Automated Testing
- Write automated tests for the following scenarios:
  - **Valid registration**
  - **Duplicate email**
  - **Missing parameters**

### 3. Scalable Email Sending
- Implement a queue to manage email sending.

### 4. Monitoring and Metrics
- Add metrics to monitor the behavior of the endpoint.
