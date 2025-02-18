## **Endpoint: `/enterprise_contact_us`**

## **1. Details**
- **Endpoint**: `/enterprise_contact_us`
- **Method**: `POST`
- **Authentication**: Valid jwt Token.
- **Purpose**: Allow logged users to contact the team for enterprise price solutions.

---

## **2. Accepted Parameters**
The request body must be in JSON format:

| **Parameter**         | **Type**  | **Required** | **Description**                        |
|---------------        |----------|--------------|-----------------------------------------|
| `name`                | String   | Yes          | The user's name for contact             |
| `title`               | String   | Yes          | The user's title in the company         |
| `company`             | String   | Yes          | The user's company.                     |
| `website`             | String   | Yes          | The user's company website.             |
| `location`            | String   | Yes          | The user's location.                    |
| `number_of_properties`   | Int   | Yes          | The number of properties                |
| `project_description`    | String   | Yes          | The user's idea.                     |


### **Example Request**
```
POST /enterprise_contact_us
Content-Type: application/json

{
  "name": "user@example.com",
  "title": "CEO"
  "company": "Meta"
  "website": "meta.com"
  "location": "London, UK"
  "number_of_properties": "10"
  "project_description": "Make money"

}
```

---

## **6. Endpoint Responses**
### **Success**
- **HTTP Status**: `201 OK`
- **Body**:
```
{
  "status": "success",
  "code": 200,
  "data": {
    "message": "Request Contact succesfully sent.",
  }
}
```

### **Errors**
#### **1. User Not Found**
- **HTTP Status**: `404 user not found`
- **Body**:
```
{
  "status": "error",
  "code": 404,
  "message": "User not found"
}
```
#### **2. Server Error**
- **HTTP Status**: `500 Internal Server Error`
- **Body**:
```
{
  "status": "error",
  "code": 500,
  "message": "Internal Server Error"
}
```
---

## **7. Implementation Code**
```

@v1.route('/enterprise_contact_us', methods=['POST'])
@jwt_required()
def enterprise_contact_us():

    user_ip = utils.get_client_ip(request)
    current_app.logger.info(f"{user_ip} - /enterprise_contact_us New Enterprise contact request.")

    try:
        current_user_id = UUID(get_jwt_identity())
        user = User.query.filter_by(user_id=current_user_id).first()

        if not user:
            current_app.logger.info(f"{user_ip} - /enterprise_contact_us User not found")
            return utils.jsonify_return_error("error", 404, "User not found"), 404


        # Estrai i dati dall'input JSON (tutti opzionali)
        data = request.json
        name = data.get('name', 'N/A')
        title = data.get('title', 'N/A')
        company = data.get('company', 'N/A')
        website = data.get('website', 'N/A')
        location = data.get('location', 'N/A')
        number_of_properties = data.get('number_of_properties', 'N/A')
        project_description = data.get('describe_your_project', 'N/A')

        # Formattiamo l'oggetto e il contenuto della mail
        email_subject = f"New Contact Request from {name} for Enterprise: {company}"
        email_body = f"""
        **Name:** {name}
        **Title:** {title}
        **Company:** {company}
        **Website:** {website}
        **Location:** {location}
        **Number of Properties:** {number_of_properties}
        **Project Description:** {project_description}
        """

        # Invia l'email con la funzione già esistente
        response = utils.send_generic_email(to_address=user.email, subject=email_subject, message=email_body)
        if response.status_code == 404:
            callback_refresh()
            response = utils.send_generic_email(to_address=user.email, subject=email_subject, message=email_body)


        current_app.logger.info(f"{user_ip} - /enterprise_contact_us success Enterprise contact request.")
        return utils.jsonify_return_success("success", 201, {
            "message": "Request Contact succesfully sent."
        }), 201
    
    except Exception as e:
        current_app.logger.error(f"{user_ip} - /enterprise_contact_us Internal Server Error. {e}")
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
    

```

---

## **8. Future Improvements**
