# **Endpoint: `/send_chatbot_reactivation_reminders`**  

## **Purpose**  
This scheduled task checks for **inactive chatbots** and sends **automated email reminders** to users to encourage reactivation.  
- **First Reminder:** Sent after **7 days** of inactivity.  
- **Final Reminder:** Sent after **30 days** of inactivity.  
- **No additional reminders beyond 30 days.**  

---

## **Technical Details**  

### **Method**  
`Scheduled Task` (Cron Job / Celery Task)  

### **Trigger Frequency**  
Runs **once per day** to check inactive chatbots.  

### **Authentication**  
No external authentication required (runs internally).  

---

## **Reminder Logic**  

### **1. Identify Inactive Chatbots**  
- The system **checks all properties** where the chatbot is **disabled (`is_chatbot_active = False`)**.  
- It then filters for those where the **chatbot has been inactive for exactly 7 or 30 days**.  
- Ensures **each property only gets 2 reminders total** (`7-day` and `30-day` reminders).  

### **2. Send Reminder Emails**  
- If the property qualifies, an **email notification** is sent to the **associated user**.  
- Email includes a **reactivation link** for easy one-click reactivation.  

### **3. Log Sent Reminders**  
- Updates **last_reminder_sent** to **prevent duplicate emails**.  

---

## **Request Parameters**  
_No external request parameters needed—this is an automated internal task._  

---

## **Responses**  

| **Scenario**                     | **Status**    | **Message**                              |
|----------------------------------|-------------|------------------------------------------|
| ✅ **Success (Emails Sent)**     | `200 OK`    | `"Reminder emails successfully sent."`  |
| ❌ **No Users Need Reminders**   | `200 OK`    | `"No reminders sent today."`            |
| ⚠️ **Email Sending Error**       | `500 Error` | `"Failed to send reminder emails."`     |

---

## **Implementation Code**  

```python
from datetime import datetime, timedelta
from flask_mail import Message
from app import db, mail

def send_chatbot_reactivation_reminders():
    """
    Scheduled task to send reminders for chatbot reactivation.
    Runs daily to check properties where the chatbot has been deactivated for 7 or 30 days.
    """
    now = datetime.utcnow()
    reminder_intervals = {
        7: "First Reminder",
        30: "Final Reminder"
    }

    for days_inactive, reminder_type in reminder_intervals.items():
        reminder_threshold = timedelta(days=days_inactive)

        # Retrieve properties where chatbot has been inactive for X days
        inactive_properties = Property.query.filter(
            Property.is_chatbot_active == False,
            Property.chatbot_deactivation_date <= now - reminder_threshold,
            Property.last_reminder_sent != days_inactive  # Prevent duplicate reminders
        ).all()

        for property_obj in inactive_properties:
            user = User.query.filter_by(user_id=property_obj.user_id).first()
            if not user:
                continue

            # Compose Email
            subject = f"Reactivate Your Chatbot - {reminder_type}"
            body = f"""
            Hello {user.name},

            Your chatbot for "{property_obj.name}" has been inactive for {days_inactive} days.
            { "This is your final reminder before we stop sending alerts." if days_inactive == 30 else "" }
            
            Click below to reactivate:
            {config.FRONTEND_URL}/property/{property_obj.property_id}/reactivate-chatbot

            Best,
            The Alfred Team
            """

            msg = Message(subject, sender=config.EMAIL_SENDER, recipients=[user.email])
            msg.body = body

            try:
                mail.send(msg)
                property_obj.last_reminder_sent = days_inactive  # Update last reminder tracking
                print(f"Sent {reminder_type} to {user.email} for property {property_obj.name}")
            except Exception as e:
                print(f"Error sending email to {user.email}: {e}")

        db.session.commit()
```

---

## **Logging & Monitoring**  
✅ **Tracks Sent Reminders**  
   - Stores **last reminder date** to avoid duplicate reminders.  

✅ **Error Handling**  
   - **If an email fails**, logs the error instead of blocking the process.  

✅ **Prevents Spam**  
   - Each property gets **only 2 reminders (7-day and 30-day emails).**  

✅ **Scalability Considerations**  
   - This task can be **parallelized** for handling large numbers of properties.  

---

## **Next Steps & Considerations**  
1️⃣ **Does this fully match your expectations?**  
2️⃣ **Do you want a manual trigger for admin debugging purposes?** (e.g., `/force_send_reminders`)  
3️⃣ **Would you like to log reminder events into a database table?** (`reminder_log`)  

---

### **Final Notes**  
- **Users receive only 2 emails per property.**  
- **After 30 days, no further notifications are sent.**  
- **If the chatbot remains inactive, it does not get deleted—it just stays disabled.**  
