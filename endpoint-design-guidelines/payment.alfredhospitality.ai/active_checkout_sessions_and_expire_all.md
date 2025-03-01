
# **Endpoint: `/active_checkout_sessions_and_expire_all`**

## **1. Purpose**
This TEMPORARY endpoint will be active just for the development stage. This endpoint serves to close all the active and pending checkout session in case they are not completed. It's a cleanup endpoint.

---

## **2. Technical Details**

### **Method**
`GET`

### **URL**
`/active_checkout_sessions_and_expire_all`

### **Authentication**
No authentication

---

## **3. Accepted Parameters**
None

---

## **4. Endpoint Logic**
### **1️⃣ Authentication**
- Extract `user_id` from the JWT token.
- Ensure the user exists in the database.

### **2️⃣ Validation**
- **All fields must be provided.**  
  - If a required field is missing, return `400 Bad Request` with a specific error message.
- **User has already a stripe_customer_id**
  - if the user has already a stripe_customer_id associated, this is not the right endpoint -> '409 error' with a specific error message.

### **4️⃣ Rate Limiting**
To prevent abuse:
- **3 requests per hour per user** (JWT-based).
---

## **5. Responses**

### ✅ **Success**
| **HTTP Status** | **Message** |
|-----------------|-------------|
| `200 success`   | it will return all the remaining open checkout session data. Just to check if there are still open session. This, therefore, should return a blank text |

#### **Example Response**
```json
{
  "status": "success",
  "code": 200,
}
```

---

### ❌ **Errors**

#### **1. Missing Fields**
| **HTTP Status** | **Message** |
|-----------------|-------------|

---

#### **6. Internal Server Error**
| **HTTP Status** | **Message** |
|-----------------|-------------|
| `500 Internal Server Error` | `"An unexpected error occurred."` |

#### **Example Response**
```json
{
  "status": "error",
  "code": 500,
  "message": "An unexpected error occurred."
}
```

---

## **6. Implementation Code**

```python
@v1.route('/active_checkout_sessions_and_expire_all', methods=['GET'])
def active_checkout_sessions():
    '''Endpoint per ottenere tutte le sessioni di checkout attive.'''
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /active_checkout_sessions Getting all active checkout sessions.")

    try:
        checkout_sessions = stripe.checkout.Session.list(limit=10)
        for session in checkout_sessions['data']:
            if session['status'] == 'open':
                stripe.checkout.Session.expire(session['id'])

        # session_ids = [session.id for session in checkout_sessions['data'] if checkout_sessions['data']['status'] == 'open']

        # current_app.logger.info(f"{user_ip} - /active_checkout_sessions Active checkout session_ids: {session_ids}")


        checkout_sessions = stripe.checkout.Session.list(limit=10)
        for session in checkout_sessions['data']:
            if session['status'] == 'open':
                current_app.logger.info(f"{user_ip} - /active_checkout_sessions Active checkout session_ids: {session['id']}")

        return utils.jsonify_return_success("success", 200, checkout_sessions), 200
    except Exception as e:
        current_app.logger.error(f"{user_ip} - /active_checkout_sessions Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
```


