

# **Endpoint: `/get_user`**

## **Purpose**
Retrieves the details of an authenticated user.  
Admins can fetch inactive users using `?admin=true`.

> Note: For Admin specific config, see [Defining Yourself as an Admin in the System](#-defining-yourself-as-an-admin-in-the-system) section below.

## **1. Technical Specifications**

### **Method**
`GET`

### **URL**
`/get_user`

### **Authentication**
✅ Requires a valid JWT token (Bearer Authentication).  
✅ If `?admin=true` is used, requires **admin permissions**.

---

## **2. Parameters**

### **Query Parameters**
| **Name**   | **Type** | **Required** | **Description**                           |
|------------|---------|--------------|-------------------------------------------|
| `admin`    | Boolean | No           | If `true`, allows retrieval of inactive users (admin only). |

### **Request Headers**
| **Header**       | **Required** | **Description**               |
|------------------|-------------|-------------------------------|
| `Authorization`  | ✅ Yes       | JWT Token for authentication. |

**Example Request (Standard User)**
```
GET /get_user
Authorization: Bearer <JWT_TOKEN>
```

**Example Request (Admin Fetching Inactive Users)**
```
GET /get_user?admin=true
Authorization: Bearer <JWT_TOKEN>
```

---

## **3. Flow Logic**

1️⃣ **JWT Authentication**
   - Extract `Authorization` token.
   - Validate token & extract `user_id`.

2️⃣ **Retrieve User from Database**
   - Fetch user details using `user_id`.

3️⃣ **Check `is_active` Status**
   - ✅ **Regular Users (`?admin=false` or no query param)**  
      - If `is_active = False`, return `403 Forbidden`.  
   - ✅ **Admins (`?admin=true`)**  
      - Can retrieve **both active & inactive users**.

4️⃣ **Format Response**
   - Include **all relevant fields**:
     - **Personal Users** → `name, surname, email, phone_number, billing_address`
     - **Company Users** → Additional fields: `company_name, company_vat, company_billing_address, company_phone_number, job_title`

5️⃣ **Return Success Response**  
   - `200 OK` → User data is returned.  
   - `403 Forbidden` → User is inactive & not an admin.  
   - `404 Not Found` → User does not exist.  

---

## **4. Responses**

### ✅ **Success**
| **HTTP Status** | **Message**                           |
|-----------------|---------------------------------------|
| `200 OK`        | User data successfully retrieved.     |

**Response Example (Personal User)**
```json
{
  "status": "success",
  "code": 200,
  "data": {
    "user_id": "123e4567-e89b-12d3-a456-426614174000",
    "email": "user@example.com",
    "name": "John",
    "surname": "Doe",
    "phone_number": "+1234567890",
    "billing_address": "123 Main St",
    "is_company": false,
    "is_verified": true,
    "is_active": true
  }
}
```

**Response Example (Company User)**
```json
{
  "status": "success",
  "code": 200,
  "data": {
    "user_id": "456e7890-a12b-34c5-d678-901234567890",
    "email": "ceo@business.com",
    "name": "Jane",
    "surname": "Smith",
    "phone_number": null,
    "billing_address": null,
    "is_company": true,
    "company_name": "TechCorp Ltd.",
    "company_vat": "IT987654321",
    "company_billing_address": "456 Business Rd",
    "company_phone_number": "+9876543210",
    "job_title": "CEO",
    "is_verified": true,
    "is_active": true
  }
}
```

---

### ❌ **Errors**
| **Cause**                  | **HTTP Status** | **Message**                                  |
|----------------------------|----------------|----------------------------------------------|
| User not found             | `404 Not Found` | `"User not found."`                         |
| Account is inactive        | `403 Forbidden` | `"Account is inactive. Please reactivate."` |
| Unauthorized (No Token)    | `401 Unauthorized` | `"Authorization token required."`          |

**Example Error (Inactive User, Non-Admin)**
```json
{
  "status": "error",
  "code": 403,
  "message": "Account is inactive. Please reactivate your account."
}
```

---

## **5. Updated Code Implementation**

```python
@v1.route('/get_user', methods=['GET'])
@jwt_required()
def get_user():
    """
    Retrieves the authenticated user's profile information.
    Admins can optionally retrieve inactive users using ?admin=true.
    """
    is_admin = request.args.get("admin", "false").lower() == "true"
    user_ip = utils.get_client_ip(request)
    current_user_id = get_jwt_identity()

    # Retrieve user from DB
    current_user = User.query.filter_by(user_id=current_user_id).first()

    if not current_user:
        return utils.jsonify_return_error("error", 404, "User not found"), 404

    # If not admin, block inactive users
    if not is_admin and not current_user.is_active:
        return utils.jsonify_return_error("error", 403, "Account is inactive. Please reactivate your account."), 403

    # Prepare response with company or personal fields
    data = {
        "user_id": current_user.user_id,
        "email": current_user.email,
        "name": current_user.name,
        "surname": current_user.surname,
        "phone_number": current_user.phone_number,
        "billing_address": current_user.billing_address,
        "is_company": current_user.is_company,
        "is_verified": current_user.is_verified,
        "is_active": current_user.is_active
    }

    if current_user.is_company:
        data.update({
            "company_name": current_user.company_name,
            "company_vat": current_user.company_vat,
            "company_billing_address": current_user.company_billing_address,
            "company_phone_number": current_user.company_phone_number,
            "job_title": current_user.job_title
        })

    # Return formatted response
    return utils.jsonify_return_success("success", 200, data), 200
```

---

## **6. Security Considerations**

✅ **Authentication**
   - JWT tokens **must be securely validated** before accessing user data.

✅ **Restricting Inactive Users**
   - Inactive users **cannot** access their account.
   - Only admins can retrieve their data.

✅ **Admin Query Restriction**
   - If `?admin=true` is used, the system **must check if the requester is an admin**.

✅ **Rate Limiting**
   - Prevent abuse by **limiting requests per minute**.

---

## **7. Database Considerations**
1. **Ensure `is_active` Is Checked on Login**
   - Users should **not** be able to log in if inactive.
2. **Ensure Unique Company VAT**
   - The VAT number must be **unique per company**.

---

## **8. Future Considerations**
🔹 **Audit Logs for Admin Requests**  
- Track **when & why** an admin retrieves user data.  
- Store logs in a secure table for **transparency**.

🔹 **User Account Deletion Policy**  
- Decide **whether inactive users should be automatically removed** after X months.

---

### ✅ **Final Thoughts**
With this approach, we **keep everything inside a single endpoint**, but ensure:
- 🔐 **Security** (Admins can retrieve inactive users, but regular users can't).
- 🏗️ **Scalability** (Easily extendable for future use cases).
- ✅ **Consistency** (All user details, both personal & company, are handled properly).  

---
# **Defining Yourself as an Admin in the System**  

Since you're the **only admin**, the easiest and **most controlled approach** would be to **manually set** your admin role in the database and enforce access control at the backend level.

---

## ✅ **How to Set Yourself as an Admin**  

### **1. Add an `is_admin` Boolean Field to Users**
- In your database, add a column to the **`users`** table:  
```sql
ALTER TABLE users ADD COLUMN is_admin BOOLEAN DEFAULT FALSE;
```
- Manually **set yourself as an admin** by updating your user record:
```sql
UPDATE users SET is_admin = TRUE WHERE email = 'your_admin_email@example.com';
```

---

### **2. Enforce Admin Access in Backend**
Modify **`get_user`** and other admin-restricted endpoints to check for `is_admin=True`.  

#### **Example: Allow Admins to Fetch Inactive Users**
```python
@v1.route('/get_user', methods=['GET'])
@jwt_required()
def get_user():
    """
    Retrieves authenticated user's profile.
    Admins can fetch inactive users with `?admin=true`
    """
    user_ip = utils.get_client_ip(request)
    current_user_id = get_jwt_identity()

    # Fetch the requesting user (to check if they are admin)
    requesting_user = User.query.filter_by(user_id=current_user_id).first()
    if not requesting_user:
        return utils.jsonify_return_error("error", 404, "User not found."), 404

    # If ?admin=true, only admins can fetch inactive users
    is_admin_request = request.args.get("admin", "").lower() == "true"
    if is_admin_request:
        if not requesting_user.is_admin:
            return utils.jsonify_return_error("error", 403, "Admin access required."), 403
        
        # Retrieve the target user
        target_user_id = request.args.get("user_id")
        if not target_user_id:
            return utils.jsonify_return_error("error", 400, "User ID required for admin lookup."), 400
        
        target_user = User.query.filter_by(user_id=target_user_id).first()
        if not target_user:
            return utils.jsonify_return_error("error", 404, "Target user not found."), 404
        
        return jsonify_return_success("success", 200, {
            "user_id": target_user.user_id,
            "email": target_user.email,
            "name": target_user.name,
            "surname": target_user.surname,
            "phone_number": target_user.phone_number,
            "vat_number": target_user.vat_number,
            "is_verified": target_user.is_verified,
            "is_active": target_user.is_active,
            "is_admin": target_user.is_admin
        }), 200

    # Normal User Fetch
    if not requesting_user.is_active:
        return utils.jsonify_return_error("error", 403, "Account is inactive."), 403

    # Return only their own data
    return jsonify_return_success("success", 200, {
        "user_id": requesting_user.user_id,
        "email": requesting_user.email,
        "name": requesting_user.name,
        "surname": requesting_user.surname,
        "phone_number": requesting_user.phone_number,
        "vat_number": requesting_user.vat_number,
        "is_verified": requesting_user.is_verified,
        "is_active": requesting_user.is_active,
        "is_admin": requesting_user.is_admin
    }), 200
```

---

## 🔍 **Final Outcome**
- **Regular users** → Can **only fetch their own data**.
- **Inactive users** → Cannot access their data (unless reactivated).
- **Admins (you)** → Can **retrieve data for any user**, including inactive ones, using:
  ```
  GET /get_user?admin=true&user_id=<target_user_id>
  ```
- **Security** → No way for regular users to **"pretend"** to be admins.

---

### **❓ Alternative Approach: Env Variable**
If you **don’t want to modify the database**, another method is to hardcode admin emails in **environment variables** and check them at runtime:
```python
import os

ADMIN_EMAILS = os.getenv("ADMIN_EMAILS", "").split(",")

if requesting_user.email in ADMIN_EMAILS:
    is_admin = True
```
This lets you **control admins without modifying the database**.

---

## ✅ **Conclusion: Best Option**
**Database flag (`is_admin` column)** is the most **scalable and secure** way.  
Would you like me to **update all relevant endpoints** to support this admin check? 🚀
