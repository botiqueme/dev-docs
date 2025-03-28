
### Stripe Data Model and Relationships

- **User (your platform)** → **Customer (Stripe)**  
- **Customer (Stripe)** → can have **one or more Subscriptions**  
- **Subscription (Stripe)** → has exactly **one payment method** attached (i.e. default card or payment source)  
- **Subscription** → can contain **one or more Subscription Items**, each representing a different product/plan (e.g., “Plan for Property 1,” “Plan for Property 2,” etc.)

**Why one subscription per payment method?**  
Because Stripe’s Billing system only allows one default payment method per subscription. If you want different payment methods for different products, you need multiple subscriptions.

---

### Basic Flow

1. **User Registration + First Subscription (Trial)**  
   - The user registers on your site, confirms their email, and must add a valid payment method to access their dashboard.  
   - You create a **Stripe Customer** and then create a **first Subscription** for that customer.  
   - This subscription includes a 15-day trial and is anchored to a fixed **billing cycle date** (e.g., the 1st of each month).  
   - Stripe will handle the trial period automatically. When the trial ends, Stripe charges the user on the anchor date.  

2. **Adding a New Product Mid-Cycle with the Same Payment Method**  
   - Suppose the user adds a new product on the 10th of the month, and they want to use the same payment method as the existing subscription.  
   - **Do not create a new subscription.** Instead, **add a new Subscription Item** to the existing subscription.  
   - Stripe will calculate the prorated charge for the days from the 10th to the next anchor date (the 1st). You won’t have to handle any complicated calculations yourself.

3. **Adding a New Product with a Different Payment Method**  
   - If the user wants to pay with a **different** card or payment method, you **create a separate Subscription** for that.  
   - You can anchor it to the same date (1st of the month) so everything renews on the same day, but each subscription will have its own separate invoice.  

4. **Switching Payment Methods for an Existing Product**  
   - If the user decides later to merge everything onto a single card (or a different card than before), you can **remove** the product from one subscription and **add** it to another.  
   - Stripe’s prorations come into play here:
     - When you remove a subscription item on the 10th, Stripe issues a prorated **credit** for the unused portion from the 10th to the next anchor date.  
     - When you add that same item to the other subscription on the 10th, Stripe calculates a prorated **charge** for that same time frame.  
     - These charges and credits typically balance out, ensuring the user doesn’t pay twice for the same days.

5. **Deleting a Product**  
   - If the user no longer wants to pay for a product, you simply remove the corresponding Subscription Item from the subscription.  
   - Stripe will issue a prorated credit (if you have proration enabled) for any unused portion of the billing cycle.

---

### Key Considerations

1. **Billing Cycle Anchor**  
   - When you create or update a subscription, you can set `billing_cycle_anchor` to a specific day (e.g., the 1st). This ensures every renewal date (for that subscription) always falls on that anchor day.

2. **Proration Behavior**  
   - By default, Stripe calculates prorations whenever you add or remove items from an active subscription.  
   - If you want this behavior, ensure you set `proration_behavior = "create_prorations"` (or `"always_invoice"`) in your subscription update calls.  
   - If proration is disabled, you risk double charging or not properly crediting the user when making mid-cycle changes.

3. **Invoices**  
   - Remember: each subscription produces a **separate invoice** in Stripe. If you want one consolidated invoice for multiple products, they must be in a **single subscription** using one payment method.  
   - If the user insists on different payment methods for different products, you’ll end up with multiple subscriptions (and therefore multiple invoices).

4. **Changing Payment Methods**  
   - A single subscription can have its default payment method changed at any time. All items within that subscription will then be billed to the new method.  
   - If you only want certain products billed to a new method, consider splitting or merging subscription items between subscriptions.

5. **Trials**  
   - Your 15-day trial is straightforward: set `trial_period_days = 15` (or `trial_end` to an exact date). After the trial, the user will start paying on the anchored date.  
   - Stripe automatically handles the transition from trial to paid subscription.

---

### Example Scenarios

1. **Scenario A: Single Subscription with Multiple Products**  
   - User signs up, picks a card → Creates Subscription 1 with a 15-day trial.  
   - Later adds more products but uses the same card → Add these products as subscription items to Subscription 1.  
   - All charges appear on the same invoice each month, on the 1st.

2. **Scenario B: Two Different Cards**  
   - User signs up with Card A → Subscription 1 (with items).  
   - User adds a second product but wants to pay with Card B → Create Subscription 2 for that product, anchored to the 1st.  
   - Now you have two separate monthly invoices, each tied to a different card.

3. **Scenario C: Moving a Product from Card B to Card A**  
   - The user initially pays for a product with Card B in Subscription 2.  
   - On the 10th of the month, they switch to Card A. You remove the product from Subscription 2 (proration credit) and add it to Subscription 1 (proration charge). Stripe calculates partial charges/credits so the user doesn’t double-pay.

---

### Conclusion

- **One subscription per payment method** is the simplest way to conceptualize your system if you need separate payment sources.  
- **One subscription with multiple items** is ideal if you want a single invoice covering multiple products on the same card.  
- Proration will handle any mid-cycle changes, and you can rely on Stripe to automatically compute the correct partial charges and credits.  
