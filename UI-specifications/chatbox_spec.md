# Chatbox Frontend Specification – Alfred Co.Host

## Scope
This specification defines the frontend requirements for embedding and managing the Robin chatbot inside the Alfred Co.Host platform.
The frontend is responsible only for UI, state handling, and backend integration. No chatbot logic is handled client-side.

---

## Figma design
https://www.figma.com/design/Q8TPWHAVfN1zjkBb4MYJzn?node-id=3113-5187#1598299911 and screen above of it

## Functional Breakdown & Effort Estimation

| Function | Description | Expected FE effort | Frontend Requirement (FE) | Backend Requirement (BE) | WHEN | NOTE |
|---------|-------------|--------------------|---------------------------|--------------------------|------|------|
| Persistent widget | The chatbot remains mounted and open while navigating across the app | Low (0.5 day) | Mount the component in the app shell/layout (not in individual pages). Manage `isOpen` state client-side | None | | |
| Widget open / close | Open and close the chatbot panel | Low (0.25 day) | Manage UI `isOpen` state. Store state in memory (optional: sessionStorage) | None | | |
| Chat initialization | Start a new conversation with welcome message | Low (0.25 day) | On first open or first message, request a new `thread_id` and send a hard-coded welcome message | Endpoint that generates and returns a `thread_id` | | Welcome message:<br>“Hi! 👋 I’m Robin, your AI assistant for Alfred Co.Host. I can help you with questions about the Platform. How can I help?” |
| Send message | User sends a question | Low–Medium (0.5 day) | Send message text + `thread_id`. Disable input while waiting for response | Endpoint that receives the message, associates it with the thread, and returns a response | | |
| Receive response | Display chatbot response | Low (0.25 day) | Render assistant message. Update messages state | Return response text (string) | | |
| Loading state | Show that the bot is replying | Low (0.25 day) | Manage `isLoading` state and display a “typing” indicator | None | | |
| ❌  Response error handling | Handle network or backend errors | Medium (0.5 day) | Show UI error message. Keep user message visible. Enable retry | Return coherent HTTP errors | Later | |
| ❌ Retry last message | Retry the last failed message | Medium (0.5 day) | Re-send last message using the same `thread_id` | Handle repeated/idempotent message sends | Later | |
| Clear thread | Reset the conversation and start over | Low (0.25 day) | Clear `messages[]`, delete `thread_id`, call reset endpoint | Endpoint that invalidates/closes the thread | | |
| Size down chatbox | Reduces the chatbox to avatar without ending the conversation | Medium (0.5 day) | Collapse the chat UI to an avatar/icon while keeping `thread_id` and messages in memory | None | | |
| New thread after reset | Start a new conversation after reset | Low (0.25 day) | Request a new `thread_id` on next message | Generate a new `thread_id` | | |
| ❌ Source rendering (optional) | Display documentation references | Low (0.25 day) | If present, render links under assistant message | Return `sources[]` list (optional) | Rejected | |
| ❌ Page context injection | Inform the bot about the current page | Low–Medium (0.25–0.5 day) | Send current URL/path together with the message | Use context to improve response | Later | |
| Authenticated access | Use platform user identity | Low (0.25 day) | Automatically include existing cookies/tokens | Validate user and context | Rejected | |
| Mobile responsiveness | Usable experience on mobile | Medium (0.5–1 day) | Full-screen or bottom-sheet layout, input always visible | None | | |

---

## Chatbox UI Guidelines

- The chatbox is anchored to the bottom-right of the screen.
- The chat component must be mounted at application layout level so it persists across navigation.
- When the chatbox is **open**, the main content area must adapt so that no UI elements (cards, buttons, navigation) are visually or interactively covered by the chatbox.
- The left sidebar and top header must always remain fully visible and interactive.
- The chatbox must appear visually on top of the page without blocking access to underlying UI elements.
- When the chatbox is **minimized**, it collapses into an avatar positioned in the bottom-right corner while keeping the conversation active.
- Opening, closing, or minimizing the chatbox must never reset or interrupt the current conversation.
- Z-index layering must be explicitly defined to avoid conflicts with modals, headers, or system UI elements.
- On mobile devices, the chatbox must switch to a full-screen or bottom-sheet layout with the input always visible.

---
