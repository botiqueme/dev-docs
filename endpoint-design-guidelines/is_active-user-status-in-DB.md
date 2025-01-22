### **Appendix: `is_active` Field in the Database**

This appendix provides a **clear overview** of how `is_active` functions and ensures the platform correctly handles user status changes.

#### **Purpose of `is_active`**
The `is_active` field in the **User table** is a **boolean flag** that determines whether a user has an **active or deactivated** account. It is used to **control access** to certain platform functionalities without permanently deleting user data.

---

### **Behavior and Impact**
1. **When `is_active = True` (Active User)**
   - The user **can log in** to the platform.
   - The **chatbot data is accessible** if the subscription is active.
   - The user can modify settings and manage chatbot configurations.

2. **When `is_active = False` (Deactivated User)**
   - The user **cannot log in**.
   - The chatbot **remains inactive** even if a subscription is restored.
   - Data is **preserved** for up to **6 months**, after which it may be **permanently deleted** unless reactivated.
   - A **reactivation request** allows the user to restore chatbot configurations but **does not restore the subscription**.

---

### **Database Implementation**
- `is_active` is a **boolean column** in the `users` table.
- It is **set to `True` by default** upon registration.
- **Soft Deactivation:** Instead of deleting a user’s data, we set `is_active = False` to allow for future reactivation.
- **Automatic Cleanup:** Users who remain deactivated for **6 months** may be permanently deleted.

```sql
ALTER TABLE users ADD COLUMN is_active BOOLEAN DEFAULT TRUE;
```

---

### **Data Flow Considerations**
- **Deactivation (`/deactivate_user`)**: Updates `is_active = False`.
- **Reactivation (`/reactivate_user`)**: Restores `is_active = True` but does not affect the subscription status.
- **Login (`/login`)**: Prevents login if `is_active = False`.
- **Account Deletion (`/delete_user`)**: Removes the user **permanently** (not reversible).

---

### **Security & GDPR Compliance**
- **User Control**: Users **decide whether to reactivate** their accounts.
- **Right to Erasure**: If a user requests permanent deletion, **all data is erased immediately**.
- **Temporary Data Retention**: GDPR compliance allows for **grace periods** before full data removal.

---

#### **Future Considerations**
- **Automated Expiration Policy**: Implement a scheduled job to delete inactive users after 6 months.
- **Notifications**: Send reminders before the **6-month expiration** to allow users to reactivate.
- **Admin Controls**: (Optional) Consider allowing **manual reactivation** by an administrator in specific cases.

---

