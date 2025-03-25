# Endpoint: /update_user

## 1. Details
- **URL**: `/update_user`
- **Method**:`PATCH`.
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
| `is_company`    | String    | Boolean if the user is a company or not         |
| `name`          | String    | User's first name.                              |
| `surname`       | String    | User's last name.                               |
| `phone_number`  | String    | User's phone number (validated format).         |
| `billing_address` | String  | Billing Adress                                  |
| `company_name`   | String    | User's company name.                           |
| `vat_number`    | String    | User's VAT number (validated format).           |
| `company_billing_address` | String | Company billing address                  |
| `company_phone_number`  |  String | Company phone number                      |
| `job_title`     | String     | job title in the company of the user           |



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
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /update_user Updating user info.")
    

    try:
        # Ottieni l'ID dell'utente dal token JWT
        current_user_id = UUID(get_jwt_identity())

        # Recupera l'utente dal database
        current_user = User.query.filter_by(user_id=current_user_id).first()
        
        if not current_user:
            current_app.logger.info(f"{user_ip} - /update_user User not found")

            return utils.jsonify_return_error("error", 404, "User not found"), 404
        
        # Recupera i dati dal corpo della richiesta
        data = request.get_json()

        if not data:
            current_app.logger.info(f"{user_ip} - /update_user No data provided")
            return utils.jsonify_return_error("error", 400, "No data provided"), 400
        
                # Gestione cambio tipo di utente (is_company)
        if 'is_company' in data:
            new_is_company = data['is_company']
            if new_is_company != current_user.is_company:
                current_app.logger.info(f"{user_ip} - /update_user Changing is_company status to {new_is_company}")
                current_user.is_company = new_is_company

                if new_is_company:
                    # Da individuale a aziendale: verifica campi aziendali obbligatori
                    required_company_fields = ["company_name", "vat_number", "company_billing_address", "company_phone_number", "job_title"]
                    for field in required_company_fields:
                        if field not in data or not data[field]:
                            current_app.logger.info(f"{user_ip} - /update_user {field.replace('_', ' ').capitalize()} is required for company registration.")
                            return utils.jsonify_return_error("Error", 400, f"{field.replace('_', ' ').capitalize()} is required for company registration."), 400
                else:
                    # Da aziendale a individuale: rimuovi campi aziendali
                    current_user.company_name = None
                    current_user.vat_number = None
                    current_user.company_billing_address = None
                    current_user.company_phone_number = None
                    current_user.job_title = None

        # Campi aggiornabili per tutti
        updatable_fields = ["name", "surname", "phone_number", "billing_address"]
        # Campi aggiornabili solo per aziende
        company_fields = ["company_name", "vat_number", "company_billing_address", "company_phone_number", "job_title"]

        # Aggiorna i campi comuni
        for field in updatable_fields:
            if field in data and data[field]:
                setattr(current_user, field, data[field])

        # Gestione campi aziendali solo se l'utente è aziendale
        if current_user.is_company:
            for field in company_fields:
                if field in data and data[field]:
                    setattr(current_user, field, data[field])
        
        # Salva le modifiche nel database
        try:
            db.session.commit()
        except Exception as e:
            db.session.rollback()
            current_app.logger.error(f"{user_ip} - /update_user An error occurred while updating the user: {e}")
            return utils.jsonify_return_error("error", 500, "An error occurred while updating the user"), 500


        # Prepara i dati aggiornati per la risposta
        updated_data = {
            "name": current_user.name,
            "surname": current_user.surname,
            "phone_number": current_user.phone_number,
            "billing_address": current_user.billing_address,
            "job_title": current_user.job_title,
            "is_company": current_user.is_company
        }
        
        # Aggiungi campi aziendali se applicabili
        if current_user.is_company:
            updated_data.update({
                "company_name": current_user.company_name,
                "vat_number": current_user.vat_number,
                "company_billing_address": current_user.company_billing_address,
                "company_phone_number": current_user.company_phone_number
            })

        # Restituisce una risposta di successo con i dati aggiornati
        current_app.logger.info(f"{user_ip} - /update_user User updated successfully")
        return utils.jsonify_return_success("success", 200, updated_data), 200

    except Exception as e:
        # Gestione di errori imprevisti
        current_app.logger.error(f"{user_ip} - /update_user Unexpected error: {e}")
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
