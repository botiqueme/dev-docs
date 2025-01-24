### **📌 API Endpoint Overview**
This table provides a structured overview of all the required endpoints, their functionalities, and any key design considerations.

| **Endpoint**                 | **Method** | **Authentication** | **Description** |
|-----------------------------|------------|---------------------|-----------------|
| **User Management** |  |  |  |
| `/register`                 | `POST`     | ❌ None (public)    | Registers a new user, requiring email verification. |
| `/verify_email`             | `GET`      | ❌ None (public)    | Verifies the email using a token. If `is_active = False`, reactivates the user. |
| `/resend_verification`      | `POST`     | ❌ None (public)    | Sends a new verification email if the user hasn't verified their account. |
| `/login`                    | `POST`     | ❌ None (public)    | Authenticates the user and returns JWT access and refresh tokens. |
| `/refresh_token`            | `POST`     | 🔑 JWT (refresh)   | Provides a new access token. Denied if `is_active = False`. |
| `/get_user`                 | `GET`      | 🔑 JWT (access)    | Retrieves authenticated user details. |
| `/update_user`              | `PATCH`    | 🔑 JWT (access)    | Updates user profile information. |
| `/delete_user`              | `DELETE`   | 🔑 JWT (access)    | Deletes or deactivates a user based on request type (`soft` or `hard`). |
| `/reactivate_user`          | `PATCH`    | 🔑 JWT (access)    | Reactivates a user account if it was deactivated. |
| `/reset_password_request`   | `POST`     | ❌ None (public)    | Sends a password reset email with a one-time token. |
| `/reset_password`           | `POST`     | ❌ None (public)    | Resets the user’s password using a valid token. |
| **Subscription Management** |  |  |  |
| `/get_subscription_status`  | `GET`      | 🔑 JWT (access)    | Retrieves the user's current subscription status. |
| `/update_payment_method`    | `POST`     | 🔑 JWT (access)    | Updates the payment method through Stripe. |
| `/cancel_subscription`      | `POST`     | 🔑 JWT (access)    | Cancels the subscription but allows access until the billing period ends. |
| `/webhook/stripe`           | `POST`     | 🔐 Secure Webhook  | Handles Stripe payment events and subscription changes. |
| **Property Management** |  |  |  |
| `/get_properties`           | `GET`      | 🔑 JWT (access)    | Retrieves the list of properties associated with the user (ID + name only). |
| `/get_property_data/{id}`   | `GET`      | 🔑 JWT (access)    | Retrieves full property details, including placeholders. |
| `/create_property`          | `POST`     | 🔑 JWT (access)    | Creates a new property with the provided name. |
| `/duplicate_property/{id}`  | `POST`     | 🔑 JWT (access)    | Duplicates a property with `_copy` appended to the name. |
| `/delete_property/{id}`     | `DELETE`   | 🔑 JWT (access)    | Permanently deletes a property. Requires manual name input confirmation. |
| `/update_property/{id}`     | `PATCH`    | 🔑 JWT (access)    | Modifies property details (e.g., name, placeholders, status). |
| `/toggle_property_status/{id}` | `PATCH` | 🔑 JWT (access)    | Changes property status (draft, active, inactive, archived). |
| **Chatbot & KB Management** |  |  |  |
| `/get_placeholders/{property_id}` | `GET` | 🔑 JWT (access) | Retrieves all placeholders for a specific property. |
| `/add_placeholder/{property_id}` | `POST` | 🔑 JWT (access) | Adds a new placeholder to a property. |
| `/update_placeholder/{property_id}/{placeholder_id}` | `PATCH` | 🔑 JWT (access) | Updates the value of a placeholder. |
| `/delete_placeholder/{property_id}/{placeholder_id}` | `DELETE` | 🔑 JWT (access) | Removes a placeholder from a property. |
| `/get_chatbot_status/{property_id}` | `GET` | 🔑 JWT (access) | Retrieves the deployment status of the chatbot for the property. |
| `/toggle_chatbot/{property_id}` | `PATCH` | 🔑 JWT (access) | Activates or deactivates the chatbot (requires an active subscription). |
| **Admin & System Logs (Future Consideration)** |  |  |  |
| `/get_users_admin` (future) | `GET` | 🔑 Admin Only | Retrieves all users (requires role-based access). |
| `/set_user_role/{id}` (future) | `PATCH` | 🔑 Admin Only | Updates a user's role (admin/user). |
| `/system_logs` (future) | `GET` | 🔑 Admin Only | Retrieves activity logs (e.g., property creation, payment events). |

---

### **📌 Key Considerations**
1️⃣ **User Inactivity Handling**
   - Inactive users **cannot log in, refresh tokens, or access the platform**.
   - If they reset their password, they remain inactive until they explicitly **reactivate their account**.

2️⃣ **Subscription Handling**
   - **Users with an expired subscription** can access the platform but **cannot activate a chatbot** for any property.
   - Stripe webhooks should handle automatic renewals, cancellations, and retries.

3️⃣ **Property Lifecycle**
   - **Draft**: Incomplete property setup.
   - **Active**: Chatbot live.
   - **Inactive**: Chatbot off, property available.
   - **Archived**: Property permanently removed.

4️⃣ **Rate Limits**
   - **Login**: 5 attempts per minute.
   - **Property Creation & Duplication**: Max **10 per hour**.
   - **Reactivation Attempts**: Max **3 per hour**.

