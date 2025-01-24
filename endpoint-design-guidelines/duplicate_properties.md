### **Endpoint: `/duplicate_property`**

## **Purpose**
This endpoint allows authenticated users to **duplicate an existing property** within their tenant.  
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
Requires a **valid JWT token**. The **tenant ID** is automatically extracted from the JWT.

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
- Extract the **user ID** and **tenant ID** from the **JWT token**.
- Ensure the user is **authenticated** before proceeding.

### **2. Retrieve Property**
- Find the **existing property** using:
  - `property_id`
  - `tenant_id` (to ensure it belongs to the authenticated user).
- If the property is **not found**, return **404 Not Found**.

### **3. Validate Name Uniqueness**
- Generate a **new name** by appending `_copy` to the original name.
- If another property with the **same name already exists**, return **409 Conflict**.

### **4. Duplicate Property**
- Create a **new property instance** with:
  - **Status** = `draft`
  - **Created by** = `current_user_id`
  - **Tenant ID** = extracted from JWT
- Save the **new property** in the database.

### **5. Duplicate Placeholders**
- Copy **all placeholders** from the original property, including:
  - **Standard placeholders** (name, address, email, phone, etc.)
  - **Custom placeholders** (with the "apply to all properties" flag if active)

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
    Endpoint to duplicate an existing property within the user's tenant.
    """

    user_ip = utils.get_client_ip(request)
    current_user_id = get_jwt_identity()
    tenant_id = utils.get_tenant_from_jwt()  # Extract tenant_id from JWT

    data = request.get_json()
    property_id = data.get("property_id")

    if not property_id:
        return jsonify_return_error("error", 400, "Missing property_id parameter."), 400

    # Retrieve property ensuring it belongs to the user's tenant
    original_property = Property.query.filter_by(property_id=property_id, tenant_id=tenant_id).first()

    if not original_property:
        return jsonify_return_error("error", 404, "Property not found."), 404

    # Generate new property name
    new_property_name = f"{original_property.name}_copy"

    # Check for name conflict
    if Property.query.filter_by(name=new_property_name, tenant_id=tenant_id).first():
        return jsonify_return_error("error", 409, "Property name already exists."), 409

    # Duplicate property
    new_property = Property(
        name=new_property_name,
        tenant_id=tenant_id,  # Automatically assigning the property to the correct tenant
        status="draft",
        created_by=current_user_id
    )
    db.session.add(new_property)
    db.session.commit()

    # Duplicate placeholders
    for placeholder in original_property.placeholders:
        new_placeholder = Placeholder(
            property_id=new_property.property_id,
            name=placeholder.name,
            value=placeholder.value,
            type=placeholder.type,
            is_custom=placeholder.is_custom
        )
        db.session.add(new_placeholder)

    db.session.commit()

    # Return success response
    return jsonify_return_success("success", 201, {
        "message": "Property duplicated successfully. Chatbot creation pending payment.",
        "property_id": new_property.property_id
    }), 201
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
