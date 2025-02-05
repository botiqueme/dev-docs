# **Endpoint: `/get_property_data`**  

#### **Purpose**  
This endpoint retrieves detailed information about a **specific property** belonging to a **specific user**, including all associated **placeholders** and their values.  

---

### **Technical Details**  

- **Method:** `GET`  
- **URL:** `/get_property_data`  
- **Authentication:** Requires **JWT Token**  
- **Scope:** Only returns properties that belong to the **authenticated user’s id**  

---

### **Request Parameters**  

| **Parameter**  | **Type**  | **Required** | **Description** |
|---------------|-----------|--------------|-----------------|
| `property_id` | UUID      | ✅ Yes       | The unique identifier of the property to retrieve. |
| `user`   | UUID      | ❌ No       | Derived from JWT; ensures only properties within the user's tenant are retrieved. |

#### **Example Request**  
```
GET /get_property_data?property_id=123e4567-e89b-12d3-a456-426614174000
Authorization: Bearer <JWT_TOKEN>
```

---

### **Endpoint Logic**  

1. **JWT Authentication**  
   - Extract the JWT token from the `Authorization` header.  
   - Decode the JWT and retrieve the `user_id` linked to the authenticated user.  

2. **Property Validation**  
   - Ensure the provided `property_id` belongs to the user
   - If the property does **not exist** or does **not belong to the user**, return a `404 Not Found`.  

3. **Retrieve Property Data**  
   - Fetch the **property details**, including:
     - Property **name**
     - Property **status** (`draft`, `active`, `inactive`, `archived`)
     - List of **placeholders** and their values (standard + custom)

4. **Response Handling**  
   - If the request is valid, return the **full property details**.  
---

### **Response Codes & Examples**  

#### ✅ **Success** (`200 OK`)  
```json
{
  "status": "success",
  "code": 200,
  "data": {
    "property_id": "123e4567-e89b-12d3-a456-426614174000",
    "tenant_id": "9f2345ff-3b67-41c8-9223-1a2b3c4d5e6f",
    "name": "Luxury Apartment Rome",
    "status": "active",
    "placeholders": [
      {
        "name": "wifi_password",
        "type": "text",
        "value": "MySecureWiFi123"
      },
      {
        "name": "owner_phone",
        "type": "phone",
        "value": "+39 333 1234567"
      }
    ],
  }
}
```

---

#### ❌ **Errors**  

| **Scenario**                 | **HTTP Status** | **Message**  |
|------------------------------|----------------|--------------|
| **Property Not Found**        | `404 Not Found` | `"Property not found or does not belong to your account."` |
| **Unauthorized Access**       | `401 Unauthorized` | `"Authorization token required."` |
| **Subscription Expired**      | `403 Forbidden` | `"Subscription expired. Reactivate to retrieve property data."` |

---

### **Implementation Code**
```python
@v1.route('/get_property_data', methods=['GET'])
@jwt_required()
def get_property_data():
    """
    Endpoint to retrieve detailed information about a specific property,
    including all associated placeholders and their values.
    Requires authentication.
    """
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /get_property_data Retrieving property data.")

    try:
        # Ottieni l'ID dell'utente corrente dal token JWT
        current_user_id = UUID(get_jwt_identity())
        user = User.query.filter_by(user_id=current_user_id).first()

        if not user:
            current_app.logger.info(f"{user_ip} - /get_property_data User not found")
            return utils.jsonify_return_error("error", 404, "User not found"), 404

        # Ottieni il property_id dai parametri della query string
        property_id = request.args.get("property_id")

        if not property_id:
            current_app.logger.info(f"{user_ip} - /get_property_data Missing property_id")
            return utils.jsonify_return_error("error", 400, "Missing property_id"), 400

        # Verifica se la proprietà esiste e appartiene all'utente
        property = Property.query.filter_by(user_id=user.user_id, id=UUID(property_id)).first()

        if not property:
            current_app.logger.info(f"{user_ip} - /get_property_data Property not found")
            return utils.jsonify_return_error("error", 404, "Property not found"), 404

        # **1. Recupera le feature standard e i loro valori**
        standard_features = db.session.query(
            PropertyFeature.id, 
            PropertyFeature.name, 
            PropertyFeatureValue.value
        ).join(PropertyFeatureValue, PropertyFeature.id == PropertyFeatureValue.feature_id
        ).filter(PropertyFeatureValue.property_id == property.id).all()

        # **2. Recupera le feature custom e i loro valori**
        custom_features = db.session.query(
            CustomFeature.id, 
            CustomFeature.name, 
            CustomFeatureValue.value
        ).join(CustomFeatureValue, CustomFeature.id == CustomFeatureValue.feature_id
        ).filter(CustomFeatureValue.property_id == property.id).all()

        # **3. Costruzione della lista dei placeholders**
        placeholder_list = []

        # Aggiungi le feature standard
        for feature in standard_features:
            placeholder_list.append({
                "feature_id": str(feature.feature_id),
                "name": feature.name,
                "value": feature.value
            })

        # Aggiungi le feature custom
        for feature in custom_features:
            placeholder_list.append({
                "feature_id": str(feature.feature_id),
                "name": feature.name,
                "value": feature.value
            })

        # Costruisci la risposta con i dettagli della proprietà e i suoi placeholders
        property_data = {
            "property_id": str(property.id),
            "name": property.name,
            "description": property.description,
            "placeholders": placeholder_list
        }

        current_app.logger.info(f"{user_ip} - /get_property_data success Property data retrieved")
        return utils.jsonify_return_success("success", 200, property_data), 200

    except ValueError as e:
        current_app.logger.error(f"{user_ip} - /get_property_data ValueError. {e}")
        return utils.jsonify_return_error("error", 400, f"/get_property_data ValueError {str(e)}"), 400
    except Exception as e:
        current_app.logger.error(f"{user_ip} - /get_property_data Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
    
```

---

### **Security & Best Practices**  
✅ **JWT Authentication** → Ensures only authorized users can access property data.  
✅ **Ownership Validation** → Prevents users from accessing data of other tenants.  
✅ **Rate Limiting (Optional)** → Avoids abuse of API calls.  
✅ **Logging & Monitoring** → Tracks property access requests for security audits.  

---

### **Next Steps**  
1️⃣ **Confirm if subscription data should be included.**  
2️⃣ **Implement UI/Frontend handling for inactive users.**  
3️⃣ **Ensure placeholders support all expected data types (text, media, etc.).**  
