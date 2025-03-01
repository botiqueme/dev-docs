
# **Endpoint: `/initial_checkout`**

## **1. Purpose**
This endpoint allows logged-in users checkout for the very first subscription.  

---

## **2. Technical Details**

### **Method**
`POST`

### **URL**
`/initial_checkout`

### **Authentication**
🔑 **Requires a valid JWT token.**  
Users must be authenticated to checkout.

---

## **3. Accepted Parameters**
The request body must be in **JSON format**.

| **Parameter**         | **Type**  | **Required** | **Description**                        |
|-----------------------|----------|--------------|-----------------------------------------|
| `plan_type`           | String   | ✅ Yes       | essential or premium                    |
| `billing_period`      | String   | ✅ Yes       | monthly or yearly                       |
| `promotional_code     | String   | :x: No       | User's company name.                   |


### **Example Request**
```json
{
  "plan_type": "essential",
  "billing_period": "monthly",
  "promotional_code": "DISCOUNT10",
}
```

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
| `201 success`   | `"New Stripe Customer created successfully and checkout completed"` |

#### **Example Response**
```json
{
  "status": "success",
  "code": 201,
  "message": "Request Contact successfully sent.",
  "checkout_session_id" : "<numerical_checkout_session_ID>"
}
```

---

### ❌ **Errors**

#### **1. Missing Fields**
| **HTTP Status** | **Message** |
|-----------------|-------------|
| `400 Bad Request` | `"Missing plan_type or billing_period"` |

#### **Example Response**
```json
{
  "status": "error",
  "code": 400,
  "message": ""Missing plan_type or billing_period""
}
```

---

#### **4. User Not Found**
| **HTTP Status** | **Message** |
|-----------------|-------------|
| `404 Not Found` | `"User not found."` |

#### **Example Response**
```json
{
  "status": "error",
  "code": 404,
  "message": "User not found."
}
```

---

#### **5. Rate Limit Exceeded**
| **HTTP Status** | **Message** |
|-----------------|-------------|
| `429 Too Many Requests` | `"Too many requests. Please try again later."` |

#### **Example Response**
```json
{
  "status": "error",
  "code": 429,
  "message": "Too many requests. Please try again later."
}
```

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
@v1.route('/initial_checkout', methods=['POST'])
@jwt_required()
@limiter.limit("3 per hour")  # Per-user rate limit
def initial_checkout():
    ''' Endpoint per la creazione di un nuovo Customer Stripe e per eseguire il primo processo di checkout. '''
    
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /initial_checkout Creating new Stripe Customer and starting the initial checkout process.")

    try:
        current_user_id = UUID(get_jwt_identity())
        user = User.query.filter_by(user_id=current_user_id).first()

        if not user:
            current_app.logger.info(f"{user_ip} - /initial_checkout User not found")
            return utils.jsonify_return_error("error", 404, "User not found"), 404
        
        # Verifica se l'utente ha già un Customer Stripe associato
        if user.stripe_customer_id:
            current_app.logger.info(f"{user_ip} - /initial_checkout User already has a Stripe Customer ID")
            return utils.jsonify_return_error("error", 409, "User already has a Stripe Customer ID"), 409
        
        data = request.json
        plan_type = data.get('plan_type')
        billing_period = data.get('billing_period')
        promotional_code = data.get('promotional_code')

        if not plan_type or not billing_period:
            current_app.logger.info(f"{user_ip} - /initial_checkout Missing plan_type or billing_period")
            return utils.jsonify_return_error("Bad Request", 400, "Missing plan_type or billing_period"), 400

        # Crea un nuovo customer su Stripe
        customer = stripe.Customer.create(
            name=user.name,
            email=user.email,
            address={
                'line1': str(user.company_billing_address if user.is_company else user.billing_address)
            },
            description=f"Stripe Customer for {user.email}",
        )

        # # Aggiorna l'utente con il nuovo Stripe Customer ID
        user.stripe_customer_id = customer['id']
        db.session.commit()

        # create the chekout session
        # Determina il Price ID in base a plan_type e billing_period
        price_key = f"{plan_type.upper()}_{billing_period.upper()}"
        price_id = os.environ.get(price_key)

        if not price_id:
            current_app.logger.info(f"{user_ip} - /initial_checkout Price ID not found for plan_type: {plan_type} and billing_period: {billing_period}")
            return utils.jsonify_return_error("error", 400, "Invalid plan type or billing period"), 400

        checkout_session = stripe.checkout.Session.create(
            customer=customer['id'],
            line_items=[{
            'price': price_id,
            'quantity': 1,
            }],
            mode='subscription',
            payment_method_types=['card'],
            allow_promotion_codes=True,
            payment_method_collection='always',
            # payment_intent_data={
            #     'setup_future_usage': 'off_session',
            # },
            subscription_data={
            # 'billing_cycle_anchor': utils.get_next_month_first_day_timestamp(),
            'trial_period_days': 15,
            # 'proration_behavior': 'create_prorations',
            },
            success_url='https://payment.alfredhospitalityai.com/api/v1/success?session_id={CHECKOUT_SESSION_ID}',
            cancel_url='https://payment.alfredhospitalityai.com/api/v1/pricing',
        )

        # # Salva la nuova subscription nella tabella Subscriptions
        # new_subscription = Subscription(
        #     user_id=current_user_id,
        #     stripe_customer_id=customer['id'],
        #     stripe_payment_method_id=checkout_session['payment_method'],
        #     plan_type='monthly'
        # )
        # db.session.add(new_customer)
        # db.session.commit()


        current_app.logger.info(f"{user_ip} - /initial_checkout New Stripe Customer created successfully and checkout completed")
        return utils.jsonify_return_success("success", 201, {
            "message": "New Stripe Customer created successfully and checkout completed",
            "checkout_session_id": checkout_session['id']
        }), 201
    except Exception as e:
        current_app.logger.error(f"{user_ip} - /initial_checkout Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
```

