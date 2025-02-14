# **Endpoint: `/register`**

## **Purpose**
This endpoint allows users to register as either an **individual** or a **company**, enforcing field validation dynamically based on the chosen registration type.  

- **Individuals (`is_company = False`)** must provide **personal details**.  
- **Companies (`is_company = True`)** must provide **business-related information** while some personal fields become optional.  

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
| **Field**     | **Type**  | **Required** | **Condition**                  |
|--------------|-----------|--------------|--------------------------------|
| `name`       | String   | ✅ Yes       | Always                         |
| `surname`    | String   | ✅ Yes       | Always                         |
| `email`      | String   | ✅ Yes       | Always                         |
| `confirm_email` | String | ✅ Yes       | Must match `email`            |
| `password`   | String   | ✅ Yes       | Always                         |
| `confirm_password` | String | ✅ Yes   | Must match `password`         |
| `is_company` | Boolean  | ✅ Yes       | Defines whether it's a company |

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
  "confirm_email": "john@example.com",
  "password": "SecurePass123!",
  "confirm_password": "SecurePass123!",
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
  "confirm_email": "alice@company.com",
  "password": "CompanyPass2023!",
  "confirm_password": "CompanyPass2023!",
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
| Email mismatch           | `400 Bad Request`    | "Emails do not match."                     |
| Password mismatch        | `400 Bad Request`    | "Passwords do not match."                  |
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

## **6. Implementation Code**

```python
@v1.route('/register', methods=['POST'])
@limiter.limit("3 per minute")
def register():
    data = request.get_json()
    is_company = data.get('is_company', False)

    # Common required fields
    required_fields = ["name", "surname", "email", "confirm_email", "password", "confirm_password"]
    for field in required_fields:
        if not data.get(field):
            return jsonify({"status": "error", "message": f"{field.replace('_', ' ').capitalize()} is required."}), 400

    if data.get('email') != data.get('confirm_email'):
        return jsonify({"status": "error", "message": "Emails do not match."}), 400

    if data.get('password') != data.get('confirm_password'):
        return jsonify({"status": "error", "message": "Passwords do not match."}), 400

    # Conditional validation
    if is_company:
        company_fields = ["company_name", "vat_number", "company_billing_address", "company_phone_number", "job_title"]
        for field in company_fields:
            if not data.get(field):
                return jsonify({"status": "error", "message": f"{field.replace('_', ' ').capitalize()} is required for company registration."}), 400
    else:
        personal_fields = ["billing_address", "phone_number"]
        for field in personal_fields:
            if not data.get(field):
                return jsonify({"status": "error", "message": f"{field.replace('_', ' ').capitalize()} is required for personal registration."}), 400

    # Hash password
    password_hash = bcrypt.hashpw(data['password'].encode('utf-8'), bcrypt.gensalt())

    # Create new user
    new_user = User(
        email=data['email'],
        password_hash=password_hash,
        name=data['name'],
        surname=data['surname'],
        is_verified=False,
        is_active=True,
        is_company=is_company
    )

    # Set company fields if applicable
    if is_company:
        new_user.company_name = data['company_name']
        new_user.vat_number = data['vat_number']
        new_user.company_billing_address = data['company_billing_address']
        new_user.company_phone_number = data['company_phone_number']
        new_user.job_title = data['job_title']
    else:
        new_user.billing_address = data['billing_address']
        new_user.phone_number = data['phone_number']

    # Save to database
    try:
        db.session.add(new_user)
        db.session.commit()
    except IntegrityError:
        db.session.rollback()
        return jsonify({"status": "error", "message": "Email already registered."}), 409

    # Send email verification
    token = generate_verification_token(data['email'])
    verify_url = url_for('v1.verify_email', token=token, _external=True)
    send_verification_email(data['email'], verify_url)

    return jsonify({"status": "success", "message": "User registered. Please verify your email."}), 201
```
