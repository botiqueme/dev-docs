# Endpoint: /update_user

## 1. Details
- **URL**: `/update_user`
- **Method**: `PUT` or `PATCH` (preferred for partial updates).
- **Authentication**: Requires a JWT token to identify and authorize the user.
- **Purpose**: Allows users to update their personal information.

## Behavior

### **Authentication**
- Retrieve the JWT token from the `Authorization` header.
- Decode the token to extract the `user_id`.
- Verify that the user exists in the database.

## 2. Accepted Parameters

The request body must be in JSON format. Updatable fields:

| **Field**       | **Type**  | **Description**                                 |
|-----------------|-----------|-------------------------------------------------|
| `name`          | String    | User's first name.                              |
| `surname`       | String    | User's last name.                               |
| `phone_number`  | String    | User's phone number (validated format).         |
| `company`       | String    | User's company name.                            |
| `vat_number`    | String    | User's VAT number (validated format).           |

Note: Each filed has to be validated before going into the database (frontend)

### **Update Logic**
- Update only the fields included in the request.
- Retain current values for fields not provided in the request.
- 
### **Response**
- Return a success message with updated data or an error message in case of failure.

## 3. Esempio di Richiesta

Richiesta `PATCH /update_user`

```
{
  "name": "John",
  "surname": "Doe",
  "phone_number": "+1234567890",
  "company": "Example Corp",
  "vat_number": "IT12345678901"
}
```

## 4. Responses

### Success

| HTTP Status | Message       |
|-------------|---------------|
| 200 OK      | "Success"     |


**Body:**
```
{
  "status": "success",
  "code": 200,
  "data": {
    "name": "John",
    "surname": "Doe",
    "phone_number": "+1234567890",
    "company": "Example Corp",
    "vat_number": "IT12345678901"
  }
}
```

### Errore: Authorization token required.

| HTTP Status | Messaggio                  |
|-------------|----------------------------|
| 401 Unauthorized | "Authorization token required." |

**Body:**
```
{
  "status": "error",
  "code": 401,
  "message": "Authorization token required."
}
```

### Errore: User not found.

| HTTP Status | Messaggio                  |
|-------------|----------------------------|
| 404 Not Found | "User not found."         |

**Body:**
```
{
  "status": "error",
  "code": 404,
  "message": "User not found."
}
```
### Errore: No Data Provided

| HTTP Status | Messaggio                  |
|-------------|----------------------------|
| 400 Error | "No data provided."         |

**Body:**
```
{
  "status": "error",
  "code": 400,
  "message": "No Data Provided."
}
```

## 5. Codice Aggiornato

```
@v1.route('/update_user', methods=['PATCH'])
@jwt_required()
def update_user():
    """
    Endpoint per aggiornare le informazioni dell'utente.
    Richiede autenticazione tramite JWT.
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
            return utils.jsonify_return_error("error", 400, "No data provided"), 400
        
        # Aggiorna i campi forniti nella richiesta
        if 'name' in data:
            user.name = data['name']
        if 'surname' in data:
            user.surname = data['surname']
        if 'phone_number' in data:
            user.phone_number = data['phone_number']
        if 'company' in data:
            user.company = data['company']
        if 'vat_number' in data:
            user.vat_number = data['vat_number']
        
        # Salva le modifiche nel database
        try:
            db.session.commit()
        except Exception:
            db.session.rollback()
            return utils.jsonify_return_error("error", 500, "An error occurred while updating the user"), 500

        # Prepara i dati aggiornati per la risposta
        updated_data = {
            "name": user.name,
            "surname": user.surname,
            "phone_number": user.phone_number,
            "company": user.company,
            "vat_number": user.vat_number
        }

        # Restituisce una risposta di successo con i dati aggiornati
        return utils.jsonify_return_success("success", 200, updated_data), 200
    except Exception as e:
        # Gestione di errori imprevisti
        return utils.jsonify_return_error("error", 500, "An unexpected error occurred"), 500

```

## 6. Validations to Implement
1. **Phone Number Format**:
   - Verify that `phone_number` is in a valid format (e.g., with international prefix, such as +1234567890).
2. **VAT Number**:
   - Validate that `vat_number` is correct for the respective country (e.g., for Italy, format IT12345678901).
3. **Missing Data**:
   - Ignore fields not included in the request without overwriting existing values.
     
## 7. Additional Considerations (Optional Improvements)

1. **Detailed Error Handling**:
   - Provide specific errors for each type of failure, such as data validation issues or database save errors.
2. **Rate Limiting**:
   - Limit the number of requests per IP or user to prevent abuse (e.g., 5 requests per minute).
3. **Audit Log**:
   - Create an audit log to track when and by whom user data updates were made.

## 8. Next Steps
1. Implement the endpoint according to the specifications.
2. Test the following scenarios:
   - Partial updates (e.g., only modifying the `name`).
   - Full updates with all fields.
   - Submission of invalid data (e.g., incorrect phone number format).
3. Update the API documentation to include details about this endpoint.
4. Integrate rate limiting to prevent abuse.
5. Evaluate implementing an audit log system to monitor updates.

---
