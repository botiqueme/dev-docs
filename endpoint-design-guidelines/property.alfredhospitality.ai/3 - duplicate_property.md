### **Endpoint: `/duplicate_property`**

## **Purpose**
This endpoint allows authenticated users to **duplicate an existing property**.  
- **The duplicate retains the same placeholders and values as the original.**  
- **The chatbot instance for the duplicated property is created only after payment.**  
- **The property name is automatically appended with `_copy` to avoid name conflicts.**  

---

## **Technical Specifications**

### **Method**
`POST`

### **URL**
`/duplicate_property`

### **Authentication**
Requires a **valid JWT token**. The **user ID** is automatically extracted from the JWT.

---

## **Request Parameters**

| **Parameter**  | **Type**  | **Required** | **Description** |
|---------------|-----------|--------------|-----------------|
| `property_id` | String    | Yes          | The ID of the property to duplicate. |

**Example Request:**
```json
POST /duplicate_property
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "property_id": "abc123"
}
```

---

## **Response Codes**

| **Scenario** | **HTTP Status** | **Message** |
|-------------|----------------|-------------|
| ✅ **Success** | `201 Created` | `"Property duplicated successfully. Chatbot creation pending payment."` |
| ❌ **Property Not Found** | `404 Not Found` | `"Property not found."` |
| ❌ **Duplicate Name Conflict** | `409 Conflict` | `"Property name already exists."` |
| ❌ **Missing Parameters** | `400 Bad Request` | `"Missing property_id parameter."` |
| ❌ **Rate Limit Exceeded** | `429 Too Many Requests` | `"Too many duplication attempts. Please try again later."` |

---

## **Endpoint Logic**

### **1. JWT Authentication**
- Extract the **user ID** from the **JWT token**.
- Ensure the user is **authenticated** before proceeding.

### **2. Retrieve Property**
- Find the **existing property** using:
  - `property_id`
  - `user_id` (to ensure it belongs to the authenticated user).
- If the property is **not found**, return **404 Not Found**.

### **3. Validate Name Uniqueness**
- Generate a **new name** by appending `_copy` to the original name.
- If another property with the **same name already exists**, return **409 Conflict**.

### **4. Duplicate Property**
- Create a **new property instance**
- Save the **new property** in the database.

### **5. Duplicate Placeholders**
- Copy **all placeholders** from the original property, including:
  - **Standard placeholders VALUES** (name, address, email, phone, etc.)
  - **Custom placeholders VALUES** (with the "apply to all properties" flag if active)

### **6. Commit & Return Success**
- Save the new property and placeholders.
- Return a **201 Created** response.

---

## **Implementation Code**
```python
@v1.route('/duplicate_property', methods=['POST'])
@jwt_required()
@limiter.limit("10 per hour")
def duplicate_property():
    """
    Endpoint to duplicate an existing property, including its feature values.
    Requires authentication.
    """
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /duplicate_property Duplicating property.")

    try:
        # Ottieni l'ID dell'utente corrente dal token JWT
        current_user_id = UUID(get_jwt_identity())
        user = User.query.filter_by(user_id=current_user_id).first()

        if not user:
            current_app.logger.info(f"{user_ip} - /duplicate_property User not found")
            return utils.jsonify_return_error("error", 404, "User not found"), 404

        data = request.get_json()
        property_id = data.get("property_id")

        if not property_id:
            current_app.logger.info(f"{user_ip} - /duplicate_property Missing property_id")
            return utils.jsonify_return_error("error", 400, "Missing property_id"), 400

        # Verifica se la proprietà esiste e appartiene all'utente
        original_property = Property.query.filter_by(user_id=user.user_id, id=UUID(property_id)).first()

        if not original_property:
            current_app.logger.info(f"{user_ip} - /duplicate_property Property not found")
            return utils.jsonify_return_error("error", 404, "Property not found"), 404
        
        copy_name = f"{original_property.name}_copy"
        # check for duplicate properties name
        property_with_same_name = Property.query.filter_by(user_id=user.user_id, name=copy_name).first()

        if property_with_same_name:
            current_app.logger.info(f"{user_ip} - /duplicate_property A property with this name already exists")
            return utils.jsonify_return_error("error", 409, "A property with this name already exists"), 409

        # Crea la nuova proprietà duplicata
        duplicated_property = Property(
            user_id=user.user_id,
            name=f"{original_property.name}_copy",
            description=f"COPY {original_property.description}"
        )

        db.session.add(duplicated_property)
        db.session.commit()  # Commit per ottenere l'ID della nuova proprietà

        # **1. Duplica i valori delle caratteristiche standard (`PropertyFeatureValue`)**
        original_feature_values = PropertyFeatureValue.query.filter(
            PropertyFeatureValue.property_id == original_property.id
        ).all()

        for feature_value in original_feature_values:
            duplicated_feature_value = PropertyFeatureValue(
                property_id=duplicated_property.id,  # Associa alla nuova proprietà
                feature_id=feature_value.feature_id,  # Mantiene lo stesso feature_id
                value=feature_value.value  # Copia il valore esistente
            )
            db.session.add(duplicated_feature_value)

        # **2. Duplica i valori delle caratteristiche personalizzate (`CustomFeatureValue`)**
        original_custom_feature_values = CustomFeatureValue.query.filter(
            CustomFeatureValue.property_id == original_property.id
        ).all()

        for custom_feature_value in original_custom_feature_values:
            duplicated_custom_feature_value = CustomFeatureValue(
                custom_feature_id=custom_feature_value.custom_feature_id,  # Mantiene lo stesso custom_feature_id
                property_id=duplicated_property.id,  # Associa alla nuova proprietà
                value=custom_feature_value.value  # Copia il valore esistente
            )
            db.session.add(duplicated_custom_feature_value)

        db.session.commit()

        current_app.logger.info(f"{user_ip} - /duplicate_property success Property duplicated")
        return utils.jsonify_return_success("success", 201, {
            "message": "Property duplicated successfully."
        }), 201

    except ValueError as e:
        current_app.logger.error(f"{user_ip} - /duplicate_property ValueError. {e}")
        return utils.jsonify_return_error("error", 400, f"/duplicate_property ValueError {str(e)}"), 400
    except Exception as e:
        current_app.logger.error(f"{user_ip} - /duplicate_property Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
```

---

## **Security & Best Practices**
✅ **No Tenant Spoofing**: Users **cannot specify a different `tenant_id`**—it's **enforced via JWT**.  
✅ **Name Collision Prevention**: The new property name **always** gets `_copy` appended.  
✅ **Rate Limiting**: Users **cannot duplicate more than 10 properties per hour**.  
✅ **Only Duplicates Owned Properties**: Prevents copying properties that don’t belong to the user.  
✅ **Chatbot Creation Deferred**: The chatbot instance is **only created after payment**, reducing unnecessary system load.  

---

## **Next Steps**
- Ensure **all `property`-related endpoints extract `tenant_id` from JWT**, not the request body.  
- Implement the **payment logic** to trigger **chatbot instance creation** after property activation.  
- Test **rate limits** to avoid abuse of property duplication.  
