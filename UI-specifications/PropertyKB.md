## Property KB: placeholder UI dynamics

Let's clarify how placeholder UI may evolve. Below the [FE and BE tech spec](#technical-specification-frontend-and-backend-handling-for-editable-placeholders)

Placeholders may be of three types types:

### Standard textual

The label and description are given by us and it's unchangeable, the value is changeable.

![standard textual](./images/standard%20textual.png)

When user starts modifying it, it turns red to show modification occurred. It stays like this until saving.

![standard textual modified](./images/standard%20textual%20modified.png)

> Note: the arrow on the right retrieves the last saved value.


### Standard media 

The label and description are given by us and it's unchangeable, the value can is changeable

![standard media](./images/standard%20media.png)

When clicking camera or video icon, opens file system. The other icons save or delete the media.

> IMPORTANT: When media is uploaded, the field goes in modified mode, that is, with red border
> ![standard media uploaded](./images/standard%20media%20uploaded.png)
> If the file is heavier than 25Mb, can the value turn red as well?

### Custom textual or media (based on the user interaction). 

Appears when user clocks on **Add a placeholder**. The user can customize:
    - label, 
    - description
    - value, and the value can be either textual or media based on the user input

![custom placeholder not changed](./images/custom%20placeholder%20not%20changed.png)

> Note ON LABEL: when user clicks on the modify icon next to the label, it shows a dropdown that looks in all custom labels already set (this, to avoid doubled custom placeholders)
> ![custom label search](./images/custom%20label%20search.png)

> Note ON DESCRIPTION: when user clicks on the modify icon next to the description, the description becomes editable
> ![editable description](./images/custom%20description.png)

> IF USER ENTERS TEXT VALUE: when a value is set, the appearance is the same as the one of standard textual placeholders when modified

> IF USER ENTERS MEDIA VALUE: when a value is set, the appearance is the same as the one of standard media placeholders when modified


## Save buttons
- 1 at the box level, enriched by a saving progress signal:
![saving progress](./images/saving%20progress.png)

- 1 at the page level


> IMPORTANT: see modals in the Figma document

# Technical Specification: Frontend and Backend Handling for Editable Placeholders

This document outlines the behavior, interactions, and data management for editable fields within a thematic box. The goal is to provide a relaxed, controlled user experience that clearly indicates when a field has been modified ("dirty"), allows for rollback to the previous value, and supports saving either inline or globally at the box level.

The specification is intended for UI/Interaction Designers, Frontend Developers, and Backend Developers.

---

## 1. Objectives

- **Simplified User Experience:**  
  Users can modify an already populated field, with clear visual feedback (e.g., red border) indicating that the field has been modified (dirty state). Users can choose to confirm the change by saving or revert it using a rollback action.

- **Control & Flexibility:**  
  - **Inline Save (Optional):** An icon/button next to each field allows users to immediately save a modification.  
  - **Global Save (Box-Level):** A single save button at the box level commits all pending (dirty) changes in that box.  
  - **Rollback:** A rollback button (e.g., an undo icon) next to each modified field lets the user restore the previous value from a local buffer, provided the change hasn't been confirmed.

- **Finality:**  
  Once the backend confirms the update, the field’s value becomes definitive, and rollback is no longer available.

---

## 2. User Interaction Flow

1. **Opening the Box:**
   - The user expands a thematic dropdown (box) that contains multiple fields (labels with inputs).

2. **Starting the Modification:**
   - Upon clicking an already populated field, the system stores the current value in a local buffer.
   - As the user modifies the field, it enters a "dirty" state, signaled by a visual change (e.g., red border).

3. **During Editing:**
   - **Visual Indicators:**  
     Modified fields display an indicator (e.g., a red border or "pending" icon) to denote unsaved changes.
   - **Rollback Option:**  
     Each dirty field shows a rollback button. If clicked, the field resets to the value stored in the local buffer.

4. **Saving Changes:**
   - **Global Save Button (Box-Level):**  
     When the user clicks the global save button within the box, the system collects only those fields that are in a dirty state (i.e., unsaved modifications) and sends them to the backend.
   - **Inline Save (Optional):**  
     If an inline save icon is provided next to a field, that field can be saved immediately. The global save button will ignore fields already confirmed.

5. **Confirmation:**
   - The backend processes the update request.  
   - Upon receiving a positive confirmation, the frontend updates the field:
     - The dirty state is cleared.
     - Visual feedback (e.g., a spinner changes to a checkmark, or the red border is removed) indicates that the new value is now definitive.
   - Once confirmed, rollback is disabled for that field.

---

## 3. Frontend Requirements

### State Management & Local Buffer
- **Local Buffer:**  
  - On field focus, store the current value in a temporary buffer to enable rollback.
  
- **Dirty State:**  
  - When the field's value is modified, mark it as "dirty" and update the visual indicator (e.g., change border color to red).

### Interaction Controls
- **Rollback Button:**  
  - Display next to each dirty field.  
  - When clicked, the field reverts to the value saved in the local buffer, and the dirty state is cleared.
  
- **Global Save Button (Box-Level):**  
  - Located within the box, it sends only the fields still marked as dirty to the backend when clicked.
  
- **Inline Save Button (Optional):**  
  - Positioned next to each field, allowing immediate save of that field’s change.  
  - The global save button will exclude fields that have already been confirmed via inline save.

### Visual Feedback
- **Modification Indicator:**  
  - Use a red border or icon to show a field is in a modified (dirty) state.
  
- **Saving Feedback:**  
  - Display a progress indicator (e.g., a spinner) while a save request is in progress.
  - Replace the spinner with a checkmark or revert the field styling to normal upon successful save.

---

## 4. Backend Requirements

### API Endpoints
- **Single Field Update:**  
  - Provide a RESTful endpoint to update an individual field’s value.
  
- **Bulk Update:**  
  - Provide a RESTful endpoint to update multiple fields in a single request, accepting a payload containing only the dirty fields.

### Data Persistence and Confirmation
- **Definitive Commit:**  
  - The new value is stored in the database only after a successful update response from the backend.
  - No multi-step versioning is maintained; the latest confirmed value becomes the definitive record.

### Error Handling
- **Error Responses:**  
  - If an update fails, return an appropriate error message.
  - The frontend should maintain the dirty state and notify the user to retry the save operation.

---

## 5. Synchronization Between Frontend and Backend

- **Data Submission:**  
  - The global save action collects only fields that are marked as dirty.
  
- **Confirmation & State Update:**  
  - The frontend waits for the backend’s confirmation before clearing the dirty state.
  - Once confirmed, visual feedback is updated (removal of red borders, spinner replaced with a checkmark) and rollback becomes unavailable for those fields.

---

## 6. Summary

This specification defines a controlled editing environment where:
- Users can modify placeholder values and immediately see visual feedback.
- The system supports both inline (per field) and global (per box) saving of modifications.
- Users can rollback changes before committing them, ensuring they are confident in the data being saved.
- The backend confirms updates, after which changes become final, maintaining a simple, non-reversible record.

This approach ensures a relaxed, user-friendly interaction while providing explicit control over data modifications.
