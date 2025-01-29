### **JWT Specification Update**

#### **Purpose**
The JWT is used for **authentication and authorization**. It must be included in the `Authorization` header for all protected endpoints.

---

## **1. JWT Structure**
A JWT consists of:
```
HEADER.PAYLOAD.SIGNATURE
```
Example:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiMTIzNCIsInRlbmFudF9pZCI6IjU2Nzg5IiwiZXhwIjoxNjc1MTI1OTQyfQ.RWaAEd_cJUKC7Ej3Nkp7iwzNNG9PLBneVuLU89qF4nY
```

---

## **2. JWT Payload (Updated Claims)**

| **Claim**      | **Type** | **Description** |
|---------------|----------|-----------------|
| `user_id`     | String   | Unique user identifier. |
| `tenant_id`   | String   | Unique identifier for the tenant (used for property management). |
| `is_active`   | Boolean  | `False` if the user is deactivated (block access). |
| `is_verified` | Boolean  | `False` if the user has not verified their email. |
| `role`        | String   | `admin` or `user` (future-proofing). |
| `exp`         | Integer  | Token expiration timestamp. |

---

## **3. Authentication & Validation**
- **Extract `tenant_id` from JWT** for all `property`-related endpoints.  
- **Block inactive users** (`is_active = False`) except for `/reactivate_user` and `/resend_verification`.  
- **Require valid JWT for all protected routes** (`Authorization: Bearer <JWT_TOKEN>`).  
- **Check expiration (`exp`)** and enforce token renewal.

---

## **4. Security Best Practices**
✅ Use HTTPS for all API calls.  
✅ Store JWT securely (prefer HTTP-only cookies over localStorage).  
✅ Implement token blacklisting on logout.  
✅ Use a strong **SECRET_KEY** and rotate periodically.  

---

## **Next Steps**
1️⃣ Ensure all endpoints correctly extract `tenant_id`.  
2️⃣ Implement role-based access (`admin/user`) when needed.  
3️⃣ Configure **automatic token refresh** for session management.  

This update aligns JWT handling with new business requirements while keeping security tight. 🚀