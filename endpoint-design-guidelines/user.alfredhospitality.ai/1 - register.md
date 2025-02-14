# **Endpoint: `/register`**

## **Purpose**
This endpoint allows users to register as either an **individual** or a **company**, enforcing dynamic validation based on the chosen registration type.

- **Individuals (`is_company = False`)** must provide **personal details**.  
- **Companies (`is_company = True`)** must provide **business-related information**, making some personal fields optional.  

---

## **1. Technical Specifications**

### **Method**
`POST`

### **URL**
`/register`

### **Authentication**
None (public).

---

## **2. Request Parameters**

### **Required Fields (Always Required)**
| **Field**         | **Type**  | **Required** | **Condition**                  |
|------------------|-----------|--------------|--------------------------------|
| `name`          | String   | ✅ Yes       | Always                         |
| `surname`       | String   | ✅ Yes       | Always                         |
| `email`         | String   | ✅ Yes       | Always                         |
| `password`      | String   | ✅ Yes       | Always                         |
| `is_company`    | Boolean  | ✅ Yes       | Defines if it’s a company      |

---

### **Conditional Fields**
| **Field**                 | **Type**  | **Required?** | **Condition**                                 |
|---------------------------|-----------|--------------|----------------------------------------------|
| `billing_address`         | String   | ✅ Yes       | Only if `is_company = False` (Individual)   |
| `phone_number`            | String   | ✅ Yes       | Only if `is_company = False` (Individual)   |
| `company_name`            | String   | ✅ Yes       | Only if `is_company = True` (Company)       |
| `vat_number`              | String   | ✅ Yes       | Only if `is_company = True` (Company)       |
| `company_billing_address` | String   | ✅ Yes       | Only if `is_company = True` (Company)       |
| `company_phone_number`    | String   | ✅ Yes       | Only if `is_company = True` (Company)       |
| `job_title`               | String   | ✅ Yes       | Only if `is_company = True` (Company)       |

---

## **3. Example Requests**

### **1️⃣ Individual Registration (`is_company = False`)**
```json
POST /register
Content-Type: application/json

{
  "name": "John",
  "surname": "Doe",
  "email": "john@example.com",
  "password": "SecurePass123!",
  "is_company": false,
  "billing_address": "123 Main St, London",
  "phone_number": "+441234567890"
}
```

---

### **2️⃣ Company Registration (`is_company = True`)**
```json
POST /register
Content-Type: application/json

{
  "name": "Alice",
  "surname": "Smith",
  "email": "alice@company.com",
  "password": "CompanyPass2023!",
  "is_company": true,
  "company_name": "Tech Solutions Ltd",
  "vat_number": "GB123456789",
  "company_billing_address": "456 Business Park, London",
  "company_phone_number": "+442012345678",
  "job_title": "CTO"
}
```

---

## **4. Response Codes**

### ✅ **Success**
| **HTTP Status** | **Message**                          |
|-----------------|---------------------------------------|
| `201 Created`   | "User registered. Please verify your email." |

### ❌ **Errors**
| **Cause**                | **HTTP Status**       | **Message**                                 |
|--------------------------|----------------------|---------------------------------------------|
| Disposable email         | `400 Bad Request`    | "Disposable emails are not allowed."       |
| Missing required fields  | `400 Bad Request`    | "Field X is required."                     |
| Duplicate email          | `409 Conflict`       | "Email already registered."                |
| Rate limit exceeded      | `429 Too Many Requests` | "Too many requests. Try again later."     |
| Database integrity issue | `500 Internal Error` | "Internal Server Error. Please contact support." |

---

## **5. Backend Logic**

### **Validation Process**
1. Retrieve `is_company` value to determine required fields.
2. **Validate common fields**:
   - Check `email`, `confirm_email`, `password`, `confirm_password`.
3. **Conditional validation**:
   - If `is_company = False`, require `billing_address` and `phone_number`.
   - If `is_company = True`, require `company_name`, `vat_number`, `company_billing_address`, `company_phone_number`, `job_title`.

### **User Creation**
1. Hash password (`bcrypt`).
2. Store user in the database.
3. Generate an **email verification token** and send an email.

---

## **6. Additional Features**

### **Rate Limiting**
- **3 registration attempts per minute per IP** using `Flask-Limiter`.

### **Database Integration**
- **`is_active` Field**:
  - Added to the `users` table as a boolean.
  - Default value: `True` (active users).
  - Used to manage account status:
    - If a user deactivates their account, set `is_active = False`.
    - If the account is reactivated, set `is_active = True`.

**Database Migration**
```sql
ALTER TABLE users ADD COLUMN is_active BOOLEAN DEFAULT TRUE;
```

---

## **7. Security Details**
- **Secure Passwords**:
  - Stored using `bcrypt` hashing.
- **Blocked Disposable Emails**:
  - Validated through a dedicated library.
- **Email Verification Token**:
  - Securely generated using `URLSafeTimedSerializer`.
- **Prevent Duplicate Registrations**:
  - Emails are unique, enforced at the database level.

---

## **8. Future Considerations**
- **Inactive Users**:
  - Ensure `/login` endpoint checks `is_active` status.
  - Add `/reactivate_user` to handle account reactivation.
- **Monitoring**:
  - Track registration and verification metrics with **Prometheus** or **Sentry**.
- **Scalability**:
  - Implement **email sending via a queue** to handle spikes in user registration.

---

## **9. Implementation Code**
```python

 @v1.route('/register', methods=['POST'])
@limiter.limit("3 per minute")
def register():
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /register Registering new user...")

    data = request.get_json()

    # Common required fields
    required_fields = ["name", "surname", "email", "password"]
    for field in required_fields:
        if not data.get(field):
            current_app.logger.info(f"{user_ip} - /register {field.replace('_', ' ').capitalize()} is required.")
            return utils.jsonify_return_error("Error", 400, f"{field.replace('_', ' ').capitalize()} is required."), 400
    
    is_company = data.get('is_company', False)

    # Conditional validation
    if is_company:
        required_company_fields = ["company_name", "vat_number", "company_billing_address", "company_phone_number", "job_title"]
        for field in required_company_fields:
            if not data.get(field):
                current_app.logger.info(f"{user_ip} - /register {field.replace('_', ' ').capitalize()} is required for company registration.")
                return utils.jsonify_return_error("Error", 400, f"{field.replace('_', ' ').capitalize()} is required for company registration."), 400
    else:
        required_individual_fields = ["billing_address", "phone_number"]
        for field in required_individual_fields:
            if not data.get(field):
                current_app.logger.info(f"{user_ip} - /register {field.replace('_', ' ').capitalize()} is required for individual registration.")
                return utils.jsonify_return_error("Error", 400, f"{field.replace('_', ' ').capitalize()} is required for individual registration."), 400

    name = data['name']
    surname = data['surname']
    email = data['email']
    password = data['password']

    phone_number = data.get('phone_number')
    vat_number = data.get('vat_number')
    billing_address = data.get('billing_address')
    company_name = data.get('company_name')
    company_billing_address = data.get('company_billing_address')
    company_phone_number = data.get('company_phone_number')
    job_title = data.get('job_title')

    # Verifica se l'email è un'email usa e getta
    disposable = is_disposable_email.check(email)
    if disposable:
        current_app.logger.info(f"{user_ip} - /register Disposable email detected")
        return utils.jsonify_return_error("Error", 400, "Disposable emails are not allowed."), 400
    

    # Hash della password
    password_hash = utils.hash_password(password)


    # Creazione nuovo utente
    new_user = User(
        email=email,
        password_hash=password_hash,
        name=name,
        surname=surname,
        is_active=True,
        is_verified=False,
        is_company=is_company
    )

    # Aggiunta dei campi opzionali solo se non sono vuoti
    if phone_number:
        new_user.phone_number = phone_number
    if vat_number:
        new_user.vat_number = vat_number
    if billing_address:
        new_user.billing_address = billing_address
    if company_name:
        new_user.company_name = company_name
    if company_billing_address:
        new_user.company_billing_address = company_billing_address
    if company_phone_number:
        new_user.company_phone_number = company_phone_number
    if job_title:
        new_user.job_title = job_title
    
    # Verifica se l'email è già registrata
    existing_user = User.query.filter_by(email=email).first()
    if existing_user:
        current_app.logger.info(f"{user_ip} - /register Email already registered")
        return utils.jsonify_return_error("Conflict", 409, "Email already registered"), 409

    # Aggiungi l'utente al database
    try:
        db.session.add(new_user)
        db.session.commit()
    except IntegrityError as e:
        db.session.rollback()
        current_app.logger.error(f"{user_ip} - /register (Integrity) Error: {e}")
        return utils.jsonify_return_error("Error", 500, "Internal (Integrity) Server Error, please contact the admin"), 500
    except Exception as e:
        db.session.rollback()
        current_app.logger.error(f"{user_ip} - /register Internal (Databse) Error: {e}")
        return utils.jsonify_return_error("Error", 500, "Internal (Databse) Server Error, please contact the admin"), 500

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
        current_app.logger.error(f"{user_ip} - /register Internal (Verification Email) Error: {e}")
        return utils.jsonify_return_error("Error", 500, "Internal (Verification Email) Server Error, please contact the admin"), 500


    if response.status_code == 200:
        current_app.logger.info(f"{user_ip} - /register User registered.")
        return utils.jsonify_return_success("success", 201, {"message": "User registered. Please verify your email."}), 201
    else:
        current_app.logger.error(f"{user_ip} - /register Internal (Generic) Server Error: {e}")
        return utils.jsonify_return_error("Error", 500, "Internal (Generic) Server Error, please contact the admin"), 500

```
