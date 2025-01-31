### **Endpoint: `/get_placeholders/<property_id>`**

#### **Purpose**
Retrieves all placeholders for a specific property. A placeholder contains information related to the placeholder, such as text fields, phone numbers, addresses, images, or videos.

---

## **1. Technical Specifications**

### **Method**
`GET`

### **URL**
`/get_placeholders/<property_id>`

### **Authentication**
🔑 **JWT (Access Token Required)**  
The request must be authenticated, and the user must have access to the specified property.

---

## **2. Request Parameters**

| **Parameter**  | **Type**  | **Required** | **Description** |
|---------------|-----------|--------------|-----------------|
| `property_id` | Path      | Yes          | The unique identifier of the property for which placeholders are being retrieved. |

**Example Request:**
```
GET /get_placeholders/123e4567-e89b-12d3-a456-426614174000
Authorization: Bearer <JWT_TOKEN>
```

---

## **3. Response**

### ✅ **Success**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `200 OK`       | "Placeholders retrieved successfully." |

#### **Example Response:**
```json
{
  "status": "success",
  "code": 200,
  "data": {
    "property_id": "123e4567-e89b-12d3-a456-426614174000",
    "placeholders": [
      {
        "id": "abc123",
        "name": "Welcome Message",
        "type": "text",
        "value": "Welcome to our property!"
      },
      {
        "id": "xyz456",
        "name": "WiFi Password",
        "type": "text",
        "value": "securepassword123"
      },
      {
        "id": "img789",
        "name": "Property Exterior",
        "type": "image",
        "value": "https://cdn.example.com/property123/front-view.jpg"
      }
    ]
  }
}
```

---

### ❌ **Errors**

#### **1️⃣ Property Not Found**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `404 Not Found` | "Property not found." |

**Example Response:**
```json
{
  "status": "error",
  "code": 404,
  "message": "Property not found."
}
```


---

#### **3️⃣ Invalid Token**
| **HTTP Status** | **Message** |
|----------------|-------------|
| `401 Unauthorized` | "Invalid or expired token." |

**Example Response:**
```json
{
  "status": "error",
  "code": 401,
  "message": "Invalid or expired token."
}
```

---

## **4. Implementation Code**
```python
@v1.route('/get_placeholders/<string:property_id>', methods=['GET'])
@jwt_required()
def get_placeholders(property_id):
    """
    Retrieves all placeholders associated with a specific property.
    Requires JWT authentication.
    """

    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /get_placeholders Retrieving placeholders for property {property_id}.")

    try:
        # Ottieni l'ID dell'utente dal token JWT
        current_user_id = get_jwt_identity()

        # Verifica se la proprietà esiste e appartiene all'utente
        property_instance = Property.query.filter_by(id=property_id, user_id=current_user_id).first()
        if not property_instance:
            current_app.logger.info(f"{user_ip} - /get_placeholders Property not found or does not belong to user.")
            return utils.jsonify_return_error("error", 404, "Property not found or does not belong to user."), 404

        # Recupera i placeholder standard associati alla proprietà
        standard_placeholders = (
            db.session.query(
                PropertyFeature.id.label('id'),
                PropertyFeature.name.label('name'),
                PropertyFeature.data_type.label('type'),
                PropertyFeatureValue.value.label('value'),
                db.literal(False).label('is_custom')
            )
            .join(PropertyFeatureValue, PropertyFeature.id == PropertyFeatureValue.feature_id)
            .filter(PropertyFeatureValue.property_id == property_id)
            .all()
        )

        # Recupera i placeholder personalizzati associati alla proprietà
        custom_placeholders = (
            db.session.query(
                CustomFeature.id.label('id'),
                CustomFeature.name.label('name'),
                CustomFeature.data_type.label('type'),
                CustomFeatureValue.value.label('value'),
                db.literal(True).label('is_custom')
            )
            .join(CustomFeatureValue, CustomFeature.id == CustomFeatureValue.feature_id)
            .filter(CustomFeatureValue.property_id == property_id, CustomFeature.user_id == current_user_id)
            .all()
        )

        # Combina i risultati
        placeholders = [
            {
                "id": str(placeholder.id),
                "name": placeholder.name,
                "type": placeholder.type,
                "value": placeholder.value,
                "is_custom": placeholder.is_custom
            }
            for placeholder in standard_placeholders + custom_placeholders
        ]

        current_app.logger.info(f"{user_ip} - /get_placeholders Placeholders retrieved successfully.")
        return utils.jsonify_return_success("success", 200, {
            "property_id": property_id,
            "placeholders": placeholders
        }), 200

    except Exception as e:
        current_app.logger.error(f"{user_ip} - /get_placeholders Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500

```

---

## **5. Security Considerations**
1. **JWT Authentication**
   - Ensures only authorized users can access their placeholders.
2. **Property Ownership Validation**
   - Prevents users from accessing placeholders of properties they do not own.
3. **Rate Limiting**
   - Implement rate limiting to avoid abuse.

---

## **6. Next Steps**
- Ensure frontend displays an appropriate message if no placeholders are found.
- Optimize database queries if properties contain many placeholders.

---

### **✅ Let me know if you want any changes before we move to the next endpoint.**
