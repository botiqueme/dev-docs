
# Endpoint: `/delete_user`

## 1. Details
- **Endpoint**: `/delete_user`
- **Method**: `DELETE`
- **Authentication**: Requires a JWT token to authenticate and authorize the user.
- **Purpose**: Allows users to delete their account and related data, with an option for temporary deactivation (soft delete) or permanent deletion (hard delete).

---

## 2. Request Parameters

| **Parameter**   | **Type**   | **Required** | **Description**                                                                 |
|-----------------|-----------|--------------|-------------------------------------------------------------------------------|
| `delete_type`   | String    | No           | The type of deletion requested: `"soft"` for temporary deactivation or `"hard"` for permanent data deletion. Defaults to `"soft"` if not specified. |

### Example Request:
```
DELETE /delete_user
Content-Type: application/json

{
  "delete_type": "soft"  # or "hard"
}
```

---

## 3. Endpoint Logic

1. **Authentication**:
   - Retrieve the JWT token from the `Authorization` header.
   - Decode the token to get the `user_id`.
   - Verify that the user exists in the database.

2. **Parameter Validation**:
   - Ensure that the `delete_type` parameter is valid. If omitted, it defaults to `"soft"`.

3. **Retrieve User**:
   - Use the `user_id` from the decoded token to fetch the user from the database.

4. **Soft Delete (Temporary Deactivation)**:
   - If the `delete_type` is `"soft"`, deactivate the account by setting `is_active = False` and retain the user’s data for 6 months.
   - The user will no longer have access to their chatbot, but their data will be kept in the system.

5. **Hard Delete (Permanent Deletion)**:
   - If the `delete_type` is `"hard"`, permanently delete the user's profile and all associated data from the database.

6. **Responses**:
   - Return a success message based on the deletion type, or an error message if something goes wrong.

---

## 4. Responses

### Success

| **HTTP Status** | **Message**                                                                |
|-----------------|----------------------------------------------------------------------------|
| `200 OK`        | "Account deactivated. Data will be retained for 6 months." (for soft delete) |
| `200 OK`        | "Account and all data have been permanently deleted." (for hard delete)    |

### Errors

| **Cause**                          | **HTTP Status**       | **Message**                                      |
|------------------------------------|-----------------------|--------------------------------------------------|
| Missing `delete_type` parameter    | `400 Bad Request`     | "Missing 'delete_type' parameter. Please specify 'soft' or 'hard'." |
| Unauthorized (missing or invalid token) | `401 Unauthorized` | "Authorization token required."                 |
| User not found                     | `404 Not Found`       | "User not found."                               |
| Invalid `delete_type` specified    | `400 Bad Request`     | "Invalid 'delete_type'. Please specify 'soft' or 'hard'."|

---

## 5. Implementation Code

```
@v1.route('/delete_user', methods=['DELETE'])
def delete_user():
    # Authentication via JWT token
    auth_header = request.headers.get('Authorization')
    if not auth_header:
        return jsonify_return_error("error", 401, "Authorization token required."), 401

    try:
        token = auth_header.split(" ")[1]
        payload = jwt.decode(token, current_app.config['SECRET_KEY'], algorithms=['HS256'])
        user_id = payload['user_id']
    except Exception:
        return jsonify_return_error("error", 401, "Invalid token"), 401

    # Retrieve the user
    user = User.query.filter_by(user_id=user_id).first()
    if not user:
        return jsonify_return_error("error", 404, "User not found"), 404

    # Retrieve data from the request
    data = request.get_json()
    delete_type = data.get('delete_type', 'soft')  # Default to 'soft' if not specified

    # Validate the 'delete_type' parameter
    if delete_type not in ['soft', 'hard']:
        return jsonify_return_error("error", 400, "Invalid 'delete_type'. Please specify 'soft' or 'hard'."), 400

    # Soft delete (temporary deactivation)
    if delete_type == 'soft':
        user.is_active = False  # Deactivate the user
        db.session.commit()
        return jsonify({
            "status": "success",
            "code": 200,
            "message": "Account deactivated. Data will be retained for 6 months."
        }), 200

    # Hard delete (permanent deletion)
    if delete_type == 'hard':
        db.session.delete(user)  # Permanently delete the user and their data
        db.session.commit()
        return jsonify({
            "status": "success",
            "code": 200,
            "message": "Account and all data have been permanently deleted."
        }), 200
```

---

## 6. Important Considerations

### **Security and Validation**:
1. **Authentication**:
   - The system requires a JWT token to verify the user's identity before allowing account deletion.
2. **Data Management**:
   - The `delete_type` parameter must be carefully checked to prevent errors and misuse.
3. **Soft Delete**:
   - The user's data is retained for 6 months and can be reactivated during this period.

---

## Appendix: Managing Data Deletion After 6 Months

### Purpose
This process involves handling the data of a deactivated user, which is retained for 6 months before being permanently deleted. This functionality aligns with GDPR requirements, allowing the user to decide whether to keep their data or delete it permanently.

### Implementation of Data Retention

1. **Soft Delete**:
   When a user requests to delete their account, their data is not immediately deleted. Instead, the data is marked as "deactivated" (soft delete). This ensures the user’s data is retained for 6 months in case they decide to return to the platform.

2. **Retention Period**:
   - The user's data is retained in a "deactivated" state for 6 months.
   - After the 6-month period, the data is automatically and permanently deleted from the platform.

3. **User Choice for Data Retention**:
   - The user can be provided with an option to choose whether they want their data to be completely deleted immediately or remain for 6 months in case they want to reactivate their account later.
   - This choice can be given during the deletion request process, with a clear explanation of the retention period.

4. **Deletion Process**:
   - After the 6-month retention period, a scheduled job (e.g., using **Celery** or a **cron job**) will run to automatically delete the user’s data. 
   - This job will ensure compliance with GDPR by permanently deleting the user data after the 6-month period.

5. **Notifications**:
   - The user will receive a confirmation email when their data is marked for deletion, with a reminder that they can request their data to be deleted immediately or retain it for 6 months.
   - If the data is not deleted immediately, the user will be notified 30 days before the 6-month retention period expires.

6. **Data Deletion Job (Scheduled Task)**:
   A cron job or Celery task will run to handle the automatic deletion of the user’s data after the 6-month retention period:
   - A SQL query will be used to select all users whose data should be deleted after the retention period:
     ```
     DELETE FROM users WHERE is_deleted = TRUE AND deactivation_date < NOW() - INTERVAL '6 months';
     ```
   - Logs will be kept to track the deletion process for auditing purposes.

7. **GDPR Compliance**:
   - This process ensures that the platform is compliant with GDPR’s "right to be forgotten" by giving the user the option to permanently delete their data or retain it for a period before removal.
   - The user can also request their data to be deleted before the 6-month period if they choose not to retain it.

