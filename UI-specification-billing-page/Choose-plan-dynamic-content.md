# Implementation Specification of secondary subscription logic

This specification defines the requirements for the activation flow of a new subscription for users who already have an active subscription, covering various payment scenarios.  
The page will be used to choose the plan, billing cycle, and payment method, and **it will dynamically display an informative message based on the user's current subscription status and the selected options.**

---

## 1. Business Logic

### Payment Scenarios

1. **Case 1: Add a MONTHLY subscription while having an active YEARLY subscription only**  
   - **Message:**  
     "Start today your monthly subscription! Next billing occurs on <next_billing_estimated+30days>. "

2. **Case 2: Add a YEARLY subscription while having an active MONTHLY subscription only**  
   - **Message:**  
     "Start today your yearly subscription! Next billing occurs on <next_billing_estimated+356 days>."

3. **Case 3: Add a MONTHLY subscription while having MONTHLY subscriptions**  
   - **Message**:  
     "Subscribe today and only pay for the days left in your active monthly billing cycle. Your next full payment will be on <date_of_monthly_renewal>."

4. **Case 4: Add a YEARLY subscription while having a YEARLY subscription**  
   - **Message**:  
     "Subscribe today and only pay for the time left in your active yearly billing cycle. Your next full payment will be on <date_of_yearly_renewal>."

5. **Case 5: Add a MONTHLY subscription while having both MONTHLY & YEARLY subscriptions**  
   - **Proposed Message**:  
     "Subscribe today and only pay for the days left in your active monthly billing cycle. Your next full payment will be on <date_of_monthly_renewal>. If you also have any active yearly subscriptions, they won't be affected."

6. **Case 6: Add a YEARLY subscription while having both MONTHLY & YEARLY subscriptions**  
   - **Proposed Message**:  
     "Subscribe today and only pay for the time left in your active yearly billing cycle. Your next full payment will be on <date_of_yearly_renewal>. If you also have any active monthly subscriptions, they won't be affected."

### Default Behavior
- On page load, **the displayed plan and billing cycle should be those last selected** (saved locally or retrieved via a dedicated endpoint).
- These default settings help determine which informative message to display by comparing the billing cycle of the user's active subscription(s) with the one selected for the new subscription.

---

## Frontend Technical Implementation Details

This section provides technical guidelines for implementing the dynamic informative message and the overall UI logic for the new subscription flow.

### 1. Data Initialization

- **Load Last Selection:**  
  On page load, retrieve the user's last-selected plan and billing cycle from local storage (or via an API call if stored server-side) and set these as the default values in the UI.

- **Fetch Active Subscription Info:**  
  Call the endpoint (e.g., `GET /subscription-info`) to retrieve details about the user's current active subscriptions. Expected data includes:
  - `currentBillingCycle` (e.g., `"monthly"`, `"yearly"`)
  - `dateOfMonthlyRenewal`
  - `dateOfYearlyRenewal`
  - *(Optional)* A flag or list indicating if the user has both monthly and yearly subscriptions.

### 2. Dynamic Informative Message Logic

Implement the dynamic message update as follows:

- **Event Handling:**  
  Attach `onChange` event listeners to the billing cycle control (toggle, radio buttons, etc.).  
  When the user selects a different billing cycle for the new subscription, trigger a function that:
  1. Reads the newly selected billing cycle.
  2. Compares it with the active subscription details fetched earlier.
  3. Determines which case applies based on the scenarios:
     - **Case 1:** New subscription is monthly and user has a yearly subscriptions but no monthly subscriptions.
     - **Case 2:** New subscription is yearly and user has monthly subscription but no yearly subscriptions.
     - **Case 3:** New subscription is monthly and user has monthly subscriptions.
     - **Case 4:** New subscription is yearly and user has yearly subscriptions.
     - **Case 5:** New subscription is monthly while user has both monthly & yearly subscriptions.
     - **Case 6:** New subscription is yearly while user has both monthly & yearly subscriptions.


- **UI Container:**  
  Reserve a section at the top of the page (or above the plan selection) for the informative message. This container should be reactive so that its content updates instantly upon billing cycle changes.

### 3. Payment Method Section

- **Existing vs. New Card:**  
  Provide a clear, toggleable option for the user to choose between:
  - Using an existing, masked payment method.
  - Entering a new card (via an inline form or modal).

- **Integration:**  
  Ensure that the selected payment method is bundled with the plan and billing cycle data when the user clicks the final Call-to-Action (e.g., "Subscribe & Go Live").

### 5. Error Handling and Feedback

- **Real-Time Validation:**  
  Validate user inputs (especially payment details) in real time, providing immediate feedback on any errors.
  
- **Error States:**  
  Display clear, contextual error messages if the subscription creation fails or if required fields are missing.

- **Success Feedback:**  
  Upon a successful subscription creation, display a confirmation message along with any next steps (e.g., confirmation email, next billing date).

---


## 3. Backend Specification

> **Note:** The specifics regarding Stripe integration are to be defined in collaboration with the backend developer. Use this section as a guideline for the necessary endpoints and logic.

### Required Endpoints
- **GET /subscription-info**  
  - **Purpose**: Retrieve the user's current subscription details.  
  - **Output**:  
    - `currentBillingCycle`: "monthly" | "yearly"  
    - `dateOfMonthlyRenewal`: The next renewal date for the monthly subscription (if applicable)  
    - `dateOfYearlyRenewal`: The next renewal date for the yearly subscription (if applicable)  
    - *(Optional)* `activeSubscriptions`: A list of active subscriptions and their billing cycles.

- **POST /create-subscription**  
  - **Purpose**: Create a new subscription based on the selected plan, billing cycle, and payment method.  
  - **Input**:  
    - `planId`: ID of the selected plan  
    - `billingCycle`: "monthly" | "yearly"  
    - `paymentMethod`: Object containing payment method details (e.g., type: "existing" or "new", along with required card info)  
    - *(Optional)* Additional fields for pro-rata calculation.
  - **Output**:  
    - Confirmation of the new subscription creation, including the `nextBillingDate` and details of any pro-rated charges.

### Pro-Rata Calculation Logic
- **Remaining Time Calculation**:  
  For the active subscription, calculate the number of days (or proportionate time) remaining in the current billing cycle.
- **Amount Determination**:  
  Compute the pro-rated amount based on the remaining time.
- **Handling Separate Billing Cycles (Case 3: Monthly → Yearly)**:  
  - Charge pro-rata for the remainder of the active monthly cycle.
  - Activate the new yearly subscription at the conclusion of the current monthly cycle, ensuring the billing cycles remain separate.

### Security Considerations
- Ensure that all sensitive payment data is handled securely and in compliance with PCI standards.
- Validate all inputs and secure the API calls using token-based authentication.
- Provide clear error messaging for any failed transactions or validation errors.
