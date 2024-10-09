---
sidebar_label: Billing and Subscription
---
# Billing and Subscription Endpoints
**Description**: This group manages billing and subscription processes for hosts, integrating with Stripe for payment operations. Authentication tokens are required for all operations. All responses follow a standard structure for consistency.

## Endpoint Response Standard
All endpoints respond using the following standardized structure:

- **Success Response**:
  ```json
  {
      "status": "success",
      "data": { /* response object */ }
  }
  ```

- **Error Response**:
  ```json
  {
      "status": "error",
      "code": 400,
      "message": "Detailed error message"
  }
  ```

## Endpoints:

### 1. POST /billing/subscribe
- **Description**: Creates a new subscription for the host using Stripe’s payment gateway.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Parameters**:
  - `user_id` (UUID, Required): Host’s unique identifier.
  - `payment_method_id` (String, Required): Payment method identifier obtained from Stripe.
  - `property_id` (UUID, Optional): Property associated with the subscription.
- **Example Request**:
  ```bash
  curl -X POST https://api.example.com/v1/billing/subscribe \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
        "user_id": "uuid-of-user",
        "payment_method_id": "stripe-payment-method-id",
        "property_id": "uuid-of-property"
      }'
  ```
- **Responses**:
  - **201 Created**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Subscription created successfully.",
            "subscription_id": "uuid-of-the-subscription"
        }
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Invalid payment method or missing data."
    }
    ```

### 2. GET /billing/subscription-status
- **Description**: Retrieves the current subscription status for a host.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Parameters**:
  - `user_id` (UUID, Required): Host’s unique identifier.
- **Example Request**:
  ```bash
  curl -X GET https://api.example.com/v1/billing/subscription-status?user_id=uuid-of-user \
  -H "Authorization: Bearer <token>"
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "subscription_id": "uuid-of-the-subscription",
            "status": "active",
            "renewal_date": "2024-12-01"
        }
    }
    ```
  - **404 Not Found**
    ```json
    {
        "status": "error",
        "code": 404,
        "message": "Subscription not found."
    }
    ```

### 3. POST /billing/update-payment-method **(inferred)**
- **Description**: Updates the host’s payment method for their active subscription.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Parameters**:
  - `user_id` (UUID, Required): Host’s unique identifier.
  - `payment_method_id` (String, Required): Updated payment method identifier from Stripe.
- **Example Request**:
  ```bash
  curl -X POST https://api.example.com/v1/billing/update-payment-method \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
        "user_id": "uuid-of-user",
        "payment_method_id": "new-stripe-payment-method-id"
      }'
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Payment method updated successfully."
        }
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Invalid payment method details."
    }
    ```

### 4. POST /billing/cancel-subscription
- **Description**: Cancels the host’s subscription for a specific property.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Parameters**:
  - `user_id` (UUID, Required): Host’s unique identifier.
  - `property_id` (UUID, Required): Property associated with the subscription to cancel.
- **Example Request**:
  ```bash
  curl -X POST https://api.example.com/v1/billing/cancel-subscription \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
        "user_id": "uuid-of-user",
        "property_id": "uuid-of-property"
      }'
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Subscription canceled successfully.",
            "valid_until": "2024-12-31"
        }
    }
    ```
  - **404 Not Found**
    ```json
    {
        "status": "error",
        "code": 404,
        "message": "Subscription not found for the provided property."
    }
    ```

### 5. GET /billing/transaction-history **(inferred)**
- **Description**: Retrieves the transaction history for a host’s account.
- **Authentication**: Requires a valid session token (`Authorization: Bearer <token>`).
- **Parameters**:
  - `user_id` (UUID, Required): Host’s unique identifier.
- **Example Request**:
  ```bash
  curl -X GET https://api.example.com/v1/billing/transaction-history?user_id=uuid-of-user \
  -H "Authorization: Bearer <token>"
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": [
            {
                "transaction_id": "uuid-of-transaction-1",
                "amount": 99.99,
                "currency": "USD",
                "date": "2024-09-01",
                "status": "completed"
            },
            {
                "transaction_id": "uuid-of-transaction-2",
                "amount": 49.99,
                "currency": "USD",
                "date": "2024-10-01",
                "status": "completed"
            }
        ]
    }
    ```
  - **404 Not Found**
    ```json
    {
        "status": "error",
        "code": 404,
        "message": "No transactions found for the user."
    }
    ```

### 6. POST /billing/webhook
- **Description**: Processes webhook events received from Stripe to manage billing updates and notifications.
- **Authentication**: No authentication required (webhook events come from Stripe directly).
- **Parameters**:
  - `event` (String, Required): Event type received from Stripe (e.g., `payment_intent.succeeded`).
  - `payload` (JSON, Required): Stripe event data payload.
- **Example Request**:
  ```bash
  curl -X POST https://api.example.com/v1/billing/webhook \
  -H "Content-Type: application/json" \
  -d '{
        "event": "payment_intent.succeeded",
        "payload": { /* Stripe event data */ }
      }'
  ```
- **Responses**:
  - **200 OK**
    ```json
    {
        "status": "success",
        "data": {
            "message": "Webhook processed successfully."
        }
    }
    ```
  - **400 Bad Request**
    ```json
    {
        "status": "error",
        "code": 400,
        "message": "Invalid webhook event data."
    }
    ```