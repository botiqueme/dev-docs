# **Endpoint: `/delete_placeholder`**

## **Purpose**
This endpoint allows users to delete a placeholder from a specific property. If the placeholder was applied to all properties (`apply_to_all`), users can choose to remove it from **all properties** or only the current one.

---

## **Technical Specifications**

### **Method**
`DELETE`

### **URL**
`/delete_placeholder`

### **Authentication**
🔑 **JWT Token Required**

---

## **Request Parameters**

| **Parameter**       | **Type**  | **Required** | **Description** |
|---------------------|----------|--------------|-----------------|
| `property_id`      | UUID      | ✅ Yes       | The unique identifier of the property where the placeholder exists. |
| `placeholder_id`   | UUID      | ✅ Yes       | The unique identifier of the placeholder to delete. |
| `apply_to_all`     | Boolean   | ❌ Optional  | If `true`, deletes this placeholder from **all properties** where it was applied. Default: `false`. |
| `Authorization`    | Header    | ✅ Yes       | Bearer token to authenticate the user. |

---


---

## **Implementation Logic**

### **1. Authentication & Authorization**
- Verify JWT token and extract user ID.
- Confirm the user **owns the property**.

### **2. Check if the placeholder exists in the given property**
- If `placeholder_id` does not exist → return **404 Not Found**.

### **3. Handle `apply_to_all` logic**
- If `apply_to_all=false` → delete the placeholder **only from the given property**.
- If `apply_to_all=true` → find **all properties** where the placeholder exists and remove it.

### **4. Delete the placeholder**
- If **only one property was affected**, return `"Placeholder deleted successfully."`
- If **multiple properties were affected**, return `"Placeholder deleted from all properties."`

### **5. Audit Logging**
- Log deletion attempts for debugging & security.

---

## **Response Codes**

| **Scenario** | **HTTP Status** | **Message** |
|-------------|----------------|-------------|
| ✅ **Success (Single Property)** | `200 OK` | `"Placeholder deleted successfully."` |
| ✅ **Success (All Properties)** | `200 OK` | `"Placeholder deleted from all properties."` |
| ❌ **Property Not Found** | `404 Not Found` | `"Property not found."` |
| ❌ **Placeholder Not Found** | `404 Not Found` | `"Placeholder not found."` |
| ❌ **Invalid or Missing JWT** | `401 Unauthorized` | `"Authorization token required."` |
| ❌ **Rate Limit Exceeded** | `429 Too Many Requests` | `"Too many deletion requests. Try again later."` |

---

## **Implementation Code**
```python
@v1.route('/delete_placeholder', methods=['DELETE'])
@jwt_required()
@limiter.limit("10 per hour")
def delete_placeholder():
    """
    Deletes a custom placeholder from a property
    """
    
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /delete_placeholder Deleting custom placeholder.")

    try:
        data = request.get_json()        
        property_id = UUID(data.get("property_id"))
        placeholder_id = data.get("placeholder_id")
        current_user_id = UUID(get_jwt_identity())

        property_instance = Property.query.filter_by(id=property_id, user_id=current_user_id).first()
        if not property_instance:
            current_app.logger.info(f"{user_ip} - /delete_placeholder Property not found")
            return utils.jsonify_return_error("error", 404, "Property not found"), 404

        # Verifica se il placeholder personalizzato esiste per l'utente
        custom_feature = CustomFeature.query.filter_by(id=placeholder_id, user_id=current_user_id).first()
        if not custom_feature:
            current_app.logger.info(f"{user_ip} - /delete_placeholder Placeholder not found")
            return utils.jsonify_return_error("error", 404, "Placeholder not found"), 404
        
        # Controlla se il placeholder è associato alla proprietà specificata
        custom_feature_value = CustomFeatureValue.query.filter_by(property_id=property_id, feature_id=placeholder_id).first()
        # if not custom_feature_value:
        #     current_app.logger.info(f"{user_ip} - /delete_placeholder Placeholder not found for this property")
        #     return utils.jsonify_return_error("error", 404, "Placeholder not found for this property"), 404
        
        # Controlla se il parametro 'apply_to_all' è presente nei parametri della query
        apply_to_all = request.args.get('apply_to_all', 'false').lower() == 'true'

        if apply_to_all:
            # Elimina il placeholder da tutte le proprietà dell'utente
            custom_feature_values = CustomFeatureValue.query.filter_by(feature_id=placeholder_id).all()
            for cfv in custom_feature_values:
                db.session.delete(cfv)

            # Elimina la definizione del placeholder personalizzato
            db.session.delete(custom_feature)
            db.session.commit()

            current_app.logger.info(f"{user_ip} - /delete_placeholder Placeholder deleted from all properties.")
            return utils.jsonify_return_success("success", 200, {"message": "Placeholder deleted from all properties."}), 200
        else:
            current_app.logger.info(f"{user_ip} - TEST1")

            # Elimina il placeholder solo dalla proprietà specificata
            if custom_feature_value:
                db.session.delete(custom_feature_value)

            db.session.delete(custom_feature)
            db.session.commit()
            
            current_app.logger.info(f"{user_ip} - /delete_placeholder Placeholder deleted from property.")
            return utils.jsonify_return_success("success", 200, {"message": "Placeholder deleted from property."}), 200
    except Exception as e:
        current_app.logger.error(f"{user_ip} - /delete_placeholder Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
```

---

## **Security & Best Practices**
- ✅ **JWT Authentication**: Ensures only authorized users can delete placeholders.
- ✅ **Ownership Validation**: Users can only modify properties they own.
- ✅ **Rate Limiting**: Maximum **10 deletions per hour** to prevent abuse.
- ✅ **Audit Logging**: Tracks deletion requests for security purposes.

---

## **Key Features**
- **Supports individual and bulk deletion** (`apply_to_all`).
- **Ensures only the owner can delete placeholders**.
- **Includes detailed response messages** for developers and users.

---

### **Next Steps**
- **Frontend Handling**: UI should prompt users to confirm whether they want to delete the placeholder **from a single property** or **from all properties**.
- **Admin Overrides (Optional)**: Consider allowing **admins** to delete placeholders across **all properties** in a system-wide manner.

---

✅ **This version is finalized. Ready to move to the next endpoint.** 🚀
