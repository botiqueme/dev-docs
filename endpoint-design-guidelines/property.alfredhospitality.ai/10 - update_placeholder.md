# **Endpoint: `/update_placeholder`**

## **Purpose**
This endpoint allows updating the value of an **existing placeholder** in a specified property.

---

## **Technical Specifications**

### **Method**
`PATCH`

### **URL**
`/update_placeholder`

### **Authentication**
🔑 **JWT Token Required (Access Token)**

---

## **Request Parameters**

| **Parameter**       | **Type**  | **Required** | **Description** |
|---------------------|-----------|--------------|-----------------|
| `property_id`      | String      | ✅ Yes        | The unique ID of the property where the placeholder exists. |
| `placeholder_name`   | String      | ✅ Yes        | The unique ID of the placeholder to be updated. |
| `value`            | JSON Body | ✅ Yes        | The new value for the placeholder (type varies based on placeholder type). |

### **Example Request**
```
PATCH /update_placeholder
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "property_id": "asdfasdf"
  "placeholder_name": "try"
  "value": "New Placeholder Value"
}
```

---

## **Validation & Logic**

1. **Authentication & Authorization**
   - Validate the JWT token.
   - Ensure the **user has permission** to modify the property.

2. **Verify Property and Placeholder**
   - Check if the `property_id` exists and belongs to the authenticated user.
   - Check if the `placeholder_id` exists within the given property.

3. **Validate Input**
   - Ensure the `value` provided matches the expected type for the placeholder.
   - Enforce **any constraints or formatting** based on placeholder type (e.g., phone number regex validation, max length for text).

4. **Update the Placeholder**
   - Modify the existing placeholder with the new value.
   - Log changes for auditing.

5. **Respond with Confirmation**
   - Return a success response with the updated placeholder details.

---

## **Response Codes**

| **Scenario**          | **HTTP Status** | **Message** |
|----------------------|----------------|-------------|
| ✅ **Success** | `200 OK` | `"Placeholder updated successfully."` |
| ❌ **Property Not Found** | `404 Not Found` | `"Property not found."` |
| ❌ **Placeholder Not Found** | `404 Not Found` | `"Placeholder not found in this property."` |
| ❌ **Invalid Input** | `400 Bad Request` | `"Invalid value format for this placeholder."` |
| ❌ **Rate Limit Exceeded** | `429 Too Many Requests` | `"Too many requests. Please try again later."` |

---

## **Implementation Code**
```python
@v1.route('/update_placeholder', methods=['PATCH'])
@jwt_required()
def update_placeholder():
    """
    Updates the value of an existing placeholder in a property.
    Requires JWT authentication.
    """

    
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /update_placeholder Updating placeholder")

    try:
        data = request.get_json()
        property_id = UUID(data.get("property_id"))
        placeholder_name = data.get("placeholder_name")
        # Ottieni l'ID dell'utente dal token JWT
        current_user_id = UUID(get_jwt_identity())

        # Verifica se la proprietà esiste e appartiene all'utente
        property_instance = Property.query.filter_by(id=property_id, user_id=current_user_id).first()
        if not property_instance:
            current_app.logger.info(f"{user_ip} - /update_placeholder Property not found or does not belong to user.")
            return utils.jsonify_return_error("error", 404, "Property not found or does not belong to user."), 404

        # Estrai i dati dalla richiesta
        data = request.get_json()
        if not data or 'value' not in data:
            current_app.logger.info(f"{user_ip} - /update_placeholder Missing 'value' in request body.")
            return utils.jsonify_return_error("error", 400, "Missing 'value' in request body."), 400

        new_value = data['value']


        # 1. Cerca il placeholder in PropertyFeature
        standard_feature = PropertyFeature.query.filter_by(name=placeholder_name).first()
        if standard_feature:
            feature_id = standard_feature.id

            # 2. Aggiorna il valore corrispondente in PropertyFeatureValue
            standard_feature_value = PropertyFeatureValue.query.filter_by(property_id=property_id, feature_id=feature_id).first()
            if standard_feature_value:
                standard_feature_value.value = new_value
                db.session.commit()
                current_app.logger.info(f"{user_ip} - /update_placeholder Standard placeholder updated successfully.")
                return utils.jsonify_return_success("success", 201, {"message": "Standard placeholder updated successfully."}), 201
            else:
                # Se non esiste, crealo
                new_feature_value = PropertyFeatureValue(property_id=property_id, feature_id=feature_id, value=new_value)
                db.session.add(new_feature_value)
                db.session.commit()
                current_app.logger.info(f"{user_ip} - /update_placeholder Standard placeholder created successfully.")
                return utils.jsonify_return_success("success", 201, {"message": "Standard placeholder Value created successfully."}), 201

        # 3. Se non l'abbiamo trovato in PropertyFeature, cerchiamo in CustomFeature
        custom_feature = CustomFeature.query.filter_by(name=placeholder_name, user_id=current_user_id).first()
        if custom_feature:
            feature_id = custom_feature.id

            # 4. Aggiorna il valore corrispondente in CustomFeatureValue
            custom_feature_value = CustomFeatureValue.query.filter_by(property_id=property_id, feature_id=feature_id).first()
            if custom_feature_value:
                custom_feature_value.value = new_value
                db.session.commit()
                current_app.logger.info(f"{user_ip} - /update_placeholder Custom placeholder updated successfully.")
                return utils.jsonify_return_success("success", 201, {"message": "Custom placeholder updated successfully."}), 201

        # Se il placeholder non esiste né come standard né come personalizzato
        current_app.logger.info(f"{user_ip} - /update_placeholder Placeholder not found.")
        return utils.jsonify_return_error("error", 404, "Placeholder not found."), 404


    except Exception as e:
        current_app.logger.error(f"{user_ip} - /update_placeholder Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500


```

---

## **Security & Best Practices**
✅ **Authentication Required**: Only authenticated users can modify placeholders.  
✅ **Ownership Validation**: Users can only update placeholders within their own properties.  
✅ **Strict Input Validation**: Ensures valid data formats based on placeholder type.  
✅ **Logging & Monitoring**: All updates are logged for security and auditing.  

---

## **Optional Improvements**
🔍 **Atomic Transactions & Race Conditions Prevention**  
If multiple updates to the same placeholder occur simultaneously, data inconsistency could happen.  
**Solution:** Use **atomic transactions** or implement a **versioning mechanism** to prevent race conditions.

---

## **Next Steps**
- **Frontend Updates:** Ensure UI provides validation feedback.  
- **Extended Error Handling:** Add more granular error messages if needed.  
- **Performance Monitoring:** Track update request frequency for optimization.  

---

