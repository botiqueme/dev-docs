Here is the **full updated specification** for the `/enterprise_contact_us` endpoint, including a **detailed overview, request handling, validation, security considerations, and implementation code**.

```markdown
# **Endpoint: `/enterprise_contact_us`**

## **1. Purpose**
This endpoint allows logged-in users to contact the Alfred team for **enterprise pricing solutions**.  
Upon submission, the system sends:
1. **A confirmation email** to the user.
2. **A notification email** to the team at `info@alfredco.host`.

---

## **2. Technical Details**

### **Method**
`POST`

### **URL**
`/enterprise_contact_us`

### **Authentication**
🔑 **Requires a valid JWT token.**  
Users must be authenticated to submit enterprise contact requests.

---

## **3. Accepted Parameters**
The request body must be in **JSON format**.

| **Parameter**         | **Type**  | **Required** | **Description**                        |
|-----------------------|----------|--------------|-----------------------------------------|
| `name`               | String   | ✅ Yes       | User's full name.                      |
| `title`              | String   | ✅ Yes       | User's job title in the company.       |
| `company`            | String   | ✅ Yes       | User's company name.                   |
| `website`            | String   | ✅ Yes       | Company's official website URL.        |
| `location`           | String   | ✅ Yes       | Company’s headquarters location.       |
| `number_of_properties` | Integer | ✅ Yes       | Number of properties owned.            |
| `project_description` | String   | ✅ Yes       | Description of the enterprise project. |

### **Example Request**
```json
{
  "name": "John Smith",
  "title": "CEO",
  "company": "Luxury Homes Inc.",
  "website": "https://luxuryhomes.com",
  "location": "London, UK",
  "number_of_properties": 15,
  "project_description": "We manage multiple properties and need a custom chatbot solution."
}
```

---

## **4. Endpoint Logic**
### **1️⃣ Authentication**
- Extract `user_id` from the JWT token.
- Ensure the user exists in the database.

### **2️⃣ Validation**
- **All fields must be provided.**  
  - If a required field is missing, return `400 Bad Request` with a specific error message.
- **Validate `website` format.**  
  - Must be a properly formatted URL.
- **Validate `number_of_properties`.**  
  - Must be a **positive integer**.
- **Limit `project_description` to 2000 characters.**

### **3️⃣ Email Handling**
📩 **Two emails are sent upon submission:**
1. **To the user** (confirmation of their request).
2. **To `info@alfredco.host`** (alerting the team about the request).

### **4️⃣ Rate Limiting**
To prevent abuse:
- **3 requests per hour per user** (JWT-based).
- **20 requests per hour per IP** (global limit for bots).

---

## **5. Responses**

### ✅ **Success**
| **HTTP Status** | **Message** |
|-----------------|-------------|
| `201 Created`   | `"Request Contact successfully sent."` |

#### **Example Response**
```json
{
  "status": "success",
  "code": 201,
  "message": "Request Contact successfully sent."
}
```

---

### ❌ **Errors**

#### **1. Missing Fields**
| **HTTP Status** | **Message** |
|-----------------|-------------|
| `400 Bad Request` | `"Missing required fields: title, website"` |

#### **Example Response**
```json
{
  "status": "error",
  "code": 400,
  "message": "Missing required fields: title, website."
}
```

---

#### **2. Invalid Website URL**
| **HTTP Status** | **Message** |
|-----------------|-------------|
| `400 Bad Request` | `"Invalid website URL format."` |

#### **Example Response**
```json
{
  "status": "error",
  "code": 400,
  "message": "Invalid website URL format."
}
```

---

#### **3. Number of Properties Not an Integer**
| **HTTP Status** | **Message** |
|-----------------|-------------|
| `400 Bad Request` | `"Number of properties must be a positive integer."` |

#### **Example Response**
```json
{
  "status": "error",
  "code": 400,
  "message": "Number of properties must be a positive integer."
}
```

---

#### **4. User Not Found**
| **HTTP Status** | **Message** |
|-----------------|-------------|
| `404 Not Found` | `"User not found."` |

#### **Example Response**
```json
{
  "status": "error",
  "code": 404,
  "message": "User not found."
}
```

---

#### **5. Rate Limit Exceeded**
| **HTTP Status** | **Message** |
|-----------------|-------------|
| `429 Too Many Requests` | `"Too many requests. Please try again later."` |

#### **Example Response**
```json
{
  "status": "error",
  "code": 429,
  "message": "Too many requests. Please try again later."
}
```

---

#### **6. Internal Server Error**
| **HTTP Status** | **Message** |
|-----------------|-------------|
| `500 Internal Server Error` | `"An unexpected error occurred."` |

#### **Example Response**
```json
{
  "status": "error",
  "code": 500,
  "message": "An unexpected error occurred."
}
```

---

## **6. Implementation Code**

```python
@v1.route('/enterprise_contact_us', methods=['POST'])
@jwt_required()
@limiter.limit("3 per hour")  # Per-user rate limit
@limiter.limit("20 per hour", key_func=get_remote_address)  # Per-IP limit
def enterprise_contact_us():
    """
    Allows logged-in users to contact the team for enterprise solutions.
    Sends:
    1. Confirmation email to the user.
    2. Notification email to info@alfredco.host.
    """
    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /enterprise_contact_us New Enterprise contact request.")

    try:
        current_user_id = UUID(get_jwt_identity())
        user = User.query.filter_by(user_id=current_user_id).first()
        if not user:
            return utils.jsonify_return_error("error", 404, "User not found"), 404

        data = request.json
        required_fields = ["name", "title", "company", "website", "location", "number_of_properties", "project_description"]
        missing_fields = [field for field in required_fields if not data.get(field)]
        if missing_fields:
            return jsonify({"status": "error", "code": 400, "message": f"Missing required fields: {', '.join(missing_fields)}"}), 400

        name = data["name"]
        title = data["title"]
        company = data["company"]
        website = data["website"]
        location = data["location"]
        project_description = data["project_description"]

        if not re.match(r'^(https?:\/\/)?([\da-z.-]+)\.([a-z.]{2,6})([\/\w .-]*)*\/?$', website):
            return jsonify({"status": "error", "code": 400, "message": "Invalid website URL format."}), 400

        try:
            number_of_properties = int(data["number_of_properties"])
            if number_of_properties <= 0:
                raise ValueError
        except ValueError:
            return jsonify({"status": "error", "code": 400, "message": "Number of properties must be a positive integer."}), 400

        admin_email_body = f"""**Name:** {name}\n**Company:** {company}\n**Website:** {website}\n**Location:** {location}\n**Properties:** {number_of_properties}\n**Project:** {project_description}"""
        user_email_body = f"Hello {name},\nWe received your request for enterprise solutions. Our team will contact you soon."

        utils.send_generic_email(user.email, "Thank you!", user_email_body)
        utils.send_generic_email("info@alfredco.host", f"New Contact: {company}", admin_email_body)

        return utils.jsonify_return_success("success", 201, {"message": "Request successfully sent."}), 201

    except Exception:
        return utils.jsonify_return_error("error", 500, "Internal Server Error"), 500
```

