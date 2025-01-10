# Endpoint: `/refresh_token`

## 1. Details
- **URL**: `/refresh_token`
- **Method**: `POST`
- **Authentication**: Requires the refresh token.
- **Purpose**: Provide a new access token when the previous one has expired.

---

## 2. Behavior

### **1. Receiving the Refresh Token**
- Retrieve the refresh token from the `Authorization` header or an HTTP-only cookie.
- 
### **2. Validating the Refresh Token**
- Decode the refresh token using the server's secret key.
- Verify:
  - Token validity (not expired).
  - Token type (`refresh`).
  - Associated user exists.
- Return an error if the token is invalid.

### **3. Generating a New Access Token**
- Create a new access token with a short expiration time.

### **4. Responding to the Frontend**
- Return the new access token.

---

## 3. Accepted Parameters
- No parameters in the request body.
- Requires the refresh token in the `Authorization` header.

Example header:
```
Authorization: Bearer <refresh_token>
```

---

## 4. Endpoint Responses

### Success
- **HTTP Status**: `200 OK`
- **Body**:
```
{
  "status": "success",
  "code": 200,
  "data": {
    "access_token": "nuovo_access_token"
  }
}
```

---

### Errors

1. **Missing Refresh Token:**:
   - **HTTP Status**: `401 Unauthorized`
   - **Body**:
```
{
  "status": "error",
  "code": 401,
  "message": "Refresh token missing."
}
```

2. **Invalid Refresh Token**:
   - **HTTP Status**: `401 Unauthorized`
   - **Body**:
```
{
  "status": "error",
  "code": 401,
  "message": "Invalid refresh token."
}
```

3. **User Not Found**:
   - **HTTP Status**: `404 Not Found`
   - **Body**:
```
{
  "status": "error",
  "code": 404,
  "message": "User not found."
}
```
4. **Refresh Token Expired**:
   - **HTTP Status**: `401 Error`
   - **Body**:
```
{
  "status": "error",
  "code": 401,
  "message": "Refresh token has expired."
}

---

## 5. Code

```
@v1.route('/refresh_token', methods=['POST'])
@jwt_required(refresh=True)
def refresh():
    """
    Endpoint per rigenerare un access token utilizzando un refresh token valido.
    """
    try:
        # Ottieni l'identità dell'utente dal refresh token
        current_user_id = get_jwt_identity()

        # Genera un nuovo access token
        new_access_token = utils.create_access_token(identity=current_user_id)

        # Prepara i dati della risposta
        data = {
            "message": "Access token refreshed successfully",
            "access_token": new_access_token
        }

        # Restituisci la risposta formattata
        return utils.jsonify_return_success("success", 200, data), 200

    except Exception as e:
        # Gestione errori generici
        return jsonify_return_error("error", 500, "An unexpected error occurred"), 500

---

## 6. Validations to Implement

1. **JWT Token**:
   - Ensure the token is valid and not expired.
   - Check that the type is `refresh`.

2. **Blacklist**:
   - Implement a blacklist to invalidate refresh tokens upon logout.

3. **Refresh Token Rotation (optional improvement)**:
   - Generate a new refresh token when the current one is used.

4. **Advanced Security (optional improvement)**:
   - Include `jti` (unique identifier) to track tokens.

---

## 7. Next Steps

1. **Implement the endpoint** following the specifications.
2. **Test**:
   - Valid requests with a working refresh token.
   - Requests with an expired or invalid token.
   - Behavior when the user is not found.
3. **Integrate the refresh token blacklist**.
4. **Update API documentation** to include details about this endpoint.
5. **Integrate token rotation (optional)** to enhance security.

---

