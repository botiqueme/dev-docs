# Endpoint: /update_password

## 1. Details
- **URL**: `/update_password`
- **Method**: `PATCH`
- **Authentication**: Requires a JWT token to identify and authorize the user.
- **Purpose**: Allows users to securely update their password.


## 2. Accepted Parameters

The request body must be in JSON format:
- **current_password** (string, mandatory): Current password
- **new_password** (string, mandatory): New password

Example:
```
{
  "current_password": "OldPass123!",
  "new_password": "NewPass456!"
}
```

## 3. Behavior

### **Authentication**
1. Retrieve the JWT token from the `Authorization` header.
2. Verify and decode the token to identify the user.

### **Validations**
1. **Presence of Fields**:
   - Ensure `current_password` and `new_password` are provided.
2. **Verify Current Password**:
   - Confirm that the provided current password matches the one stored (hashed) in the database.
3. **Validate New Password**:
   - Ensure the new password meets security criteria:
     - Minimum 8 characters.
     - At least one uppercase letter, one number, and one special character.
     - Different from the current password.

### **Update Process**
1. Generate a hash of the new password using `bcrypt`.
2. Update the hashed password in the database.

### **Response**
- Confirm the update with a success message.


## 4. Responses

### Success

| HTTP Status | Message                        |
|-------------|--------------------------------|
| 200 OK      | "Password updated successfully." |

### Errors

| Cause                    | HTTP Status       | Message                                      |
|--------------------------|-------------------|----------------------------------------------|
| Missing Parameters       | 400 Bad Request  | "Both current and new passwords are required." |
| Incorrect Current Password | 400 Bad Request | "Current password is incorrect."            |
| Invalid Token            | 401 Unauthorized | "Invalid token."                             |
| User Not Found           | 404 Not Found    | "User not found."                            |
| Internal Server Error    | 500 Error        | "Internal Server Error."                     |


## 5. Codice Aggiornato

```
@v1.route('/update_password', methods=['PATCH'])
@jwt_required()
def update_password():
    """
    Endpoint per aggiornare la password dell'utente autenticato.
    Richiede un token JWT valido.
    """
    try:
        # Ottieni l'ID dell'utente dal token JWT
        user_id = get_jwt_identity()

        # Recupera l'utente dal database
        user = User.query.get(user_id)
        if not user:
            return utils.jsonify_return_error("error", 404, "User not found"), 404

        # Recupera i dati dal corpo della richiesta
        data = request.get_json()
        if not data:
            return utils.jsonify_return_error("Bad Request", 400, "Both current and new passwords are required."), 400

        # Estrai le password dal payload
        current_password = data.get('current_password')
        new_password = data.get('new_password')

        # Verifica che tutte le password siano fornite
        if not all([current_password, new_password]):
            return utils.jsonify_return_error("Bad Request", 400, "Both current and new passwords are required."), 400
        
        # Verifica la password utilizzando bcrypt
        if not utils.verify_password(user.password_hash, current_password):  # Confronto con la password hashata
            return utils.jsonify_return_error("Bad Request", 40, "Current password is incorrect."), 400
        
        # Aggiorna la password dell'utente
        user.password_hash = utils.hash_password(new_password)

        # Salva le modifiche nel database
        db.session.commit()

        # Restituisci una risposta di successo
        return utils.jsonify_return_success("OK", 200, {"message": "Password updated successfully"}), 200

    except Exception as e:
        # Gestione degli errori imprevisti
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
```

## 6. Future Improvements

- **JWT Token Rotation** (optional improvement):
  - After a password update, invalidate current JWT tokens and require the user to log in again.
- **Email Notification** (optional improvement):
  - Send a notification to inform the user that their password has been updated.
- **Advanced Logging** (optional improvement):
  - Log password update attempts to identify suspicious activity.

## 7. Next Steps
1. Implement the endpoint according to the specifications.
2. Test the following scenarios:
   - Successful update with valid data.
   - Attempts with incorrect current passwords.
   - Validation of non-compliant new passwords.
   - Exceeding the rate limit.

