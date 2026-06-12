# Chat & Notifications — Manual Test Cases

App: https://staging.autopilotee.com (AutoPilotee Cars React frontend)

Notes:
- Chat is backed by Firebase Firestore; notifications are GraphQL (notificationService).
- Chat list/detail require an authenticated user. A chat thread only exists when there is at least one booking between a guest and a host. Sending a message requires the booking object to load (the input is gated on `booking` being present).
- The ChatPage has a "View Mode" Guest/Host toggle that filters chats/notifications by role. The role persists in localStorage.
- Selectors below come from rendered text and structure (the components use styled-components, not data-testid). Use visible text as the primary anchor for Playwright (e.g. role=button name, placeholder text).

---

### CHAT-01: Chat hub loads with View Mode toggle and Messages/Notifications tabs
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Logged in as USER1 or USER2 (any authenticated user). Use the account that has at least one booking so chat data is meaningful.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/chat
  2. Observe the top "View Mode:" control with two buttons labeled "Guest" and "Host"
  3. Observe the tab bar below it with tabs "Messages" and "Notifications"
- **Expected:**
  - Page renders without crash. "View Mode:" label visible with Guest and Host toggle buttons.
  - Two tabs render: "Messages" (active by default, pink underline) and "Notifications".
  - If there are unread messages/notifications, a pink count Badge appears on the corresponding tab (shows "99+" when over 99).
- **Source:** src/app/containers/ChatPage/index.tsx

### CHAT-02: Messages tab lists conversations or shows empty state
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** Logged in. Messages tab active.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/chat
  2. Ensure the "Messages" tab is selected
- **Expected:**
  - Header shows title "Messages" and subtitle "Chat with hosts about your bookings".
  - If the user has conversations: each row is a card showing "Booking #" + first 8 chars of bookingId, a relative timestamp (e.g. "2 hours ago"), a last-message preview (or "Say hello!"), participant name + car make/model line, and an avatar with initials. Unread rows show a pink unread badge ("9+" cap).
  - If the user has no conversations: empty state with "No conversations yet" and "You can start a conversation with a host from your booking details."
- **Source:** src/app/containers/ChatPage/ChatListPage/index.tsx

### CHAT-03: Switching View Mode filters chats by role
- **Priority:** P1  **Persona:** Host
- **Preconditions:** Logged in with an account that has BOTH guest bookings and host bookings (or at least bookings in one role). Discover which account has host bookings.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/chat
  2. Note the conversation list under default ("Guest") view mode
  3. Click the "Host" button in the View Mode toggle
- **Expected:**
  - The "Host" button becomes active (white background, pink text).
  - The conversation list re-subscribes and shows only chats where the user is the host (list may differ from Guest view; may be empty if no host chats).
  - Selection persists on reload (role is stored in localStorage).
- **Source:** src/app/containers/ChatPage/index.tsx (handleRoleChange, subscribeToUserChats with role)

### CHAT-04: Open a conversation detail
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Logged in with at least one existing conversation visible on the Messages tab.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/chat
  2. Click the first conversation card
- **Expected:**
  - URL changes to /chat/<chatId>.
  - Header shows a back arrow, an avatar with the other participant's initials, title "Booking #" + first 8 chars of bookingId, and subtitle with the other participant's name.
  - A booking card renders with the car thumbnail (or placeholder), car make+model, the booking date range "MMM dd, yyyy - MMM dd, yyyy", and the booking status in pink.
  - Existing messages render as bubbles (own messages pink on the right, others white on the left) with HH:mm timestamps.
  - A message input with placeholder "Type a message..." and a paper-plane send button are present at the bottom.
  - If the booking fails to load, "Booking not found" is shown instead.
- **Source:** src/app/containers/ChatPage/ChatDetailPage/index.tsx

### CHAT-05: Send a chat message
- **Priority:** P0  **Persona:** Guest
- **Preconditions:** Logged in. An open conversation (/chat/<chatId>) where the other participant is NOT a deleted user, and the booking has loaded.
- **Steps:**
  1. Open a conversation as in CHAT-04
  2. Click the message input (placeholder "Type a message...")
  3. Type a benign test message, e.g. "Test message from automated QA"
  4. [MUTATING] Click the paper-plane send button (or press Enter to submit the form)
- **Expected:**
  - The send button is disabled while the input is empty/whitespace and while sending.
  - On success the input clears and the new message appears as a pink bubble on the right with current HH:mm; the view auto-scrolls to bottom.
  - The other participant's unread count increments (mark-as-read happens on their side when they open the chat).
- **Source:** src/app/containers/ChatPage/ChatDetailPage/index.tsx (handleSendMessage, chatService.sendMessage)

### CHAT-06: Content moderation blocks sensitive words
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Logged in, an open conversation as in CHAT-05.
- **Steps:**
  1. In the message input, type a message containing a clearly sensitive/blocked term (executor: use a known profanity/sensitive word per backend moderation list; keep it minimal)
  2. [MUTATING] Attempt to send the message
- **Expected:**
  - A red toast appears: "Message cannot be sent. It contains sensitive words. Please revise your message." (auto-closes ~4s).
  - The message is NOT added to the thread; the input retains the typed text.
- **Source:** src/app/containers/ChatPage/ChatDetailPage/index.tsx (error handling for inappropriate content / blockedWords)

### CHAT-07: Report user modal opens from conversation header
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Logged in, open conversation where the other participant is present and NOT deleted (the flag button only shows then).
- **Steps:**
  1. Open a conversation (/chat/<chatId>)
  2. Click the flag icon button (title "Report User") in the header
- **Expected:**
  - A "Report User" modal opens, pre-targeting the other participant (reportedUserId / reportedUserName) and carrying bookingId + chatId.
  - Do NOT submit a real report unless the case is explicitly run as mutating.
- **Source:** src/app/containers/ChatPage/ChatDetailPage/index.tsx (ReportUserModal); src/app/components/chat/ReportUserModal

### CHAT-08: Deleted-participant conversation is read-only
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** A conversation where the other party has deleted their account (chat doc has guestDeleted/hostDeleted true). May not be reproducible on demand; run only if such a thread exists.
- **Steps:**
  1. Open the conversation with the deleted user
- **Expected:**
  - The header shows the backup display name or "Deleted User"; the flag/report button is hidden.
  - A yellow notice appears: "This user has deleted their account. You can view the chat history but cannot send new messages."
  - The input placeholder reads "Cannot send messages to deleted user" and is disabled (opacity 0.5); send button disabled.
- **Source:** src/app/containers/ChatPage/ChatDetailPage/index.tsx (isOtherParticipantDeleted)

### CHAT-09: Back navigation from conversation returns to chat hub
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Open conversation.
- **Steps:**
  1. Open a conversation (/chat/<chatId>)
  2. Click the back arrow in the header
- **Expected:** Navigates to /chat (the chat hub, Messages tab).
- **Source:** src/app/containers/ChatPage/ChatDetailPage/index.tsx (navigate('/chat'))

### CHAT-10: Notifications tab renders list or empty state
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** Logged in.
- **Steps:**
  1. Navigate to https://staging.autopilotee.com/chat?tab=notifications (or click the "Notifications" tab)
- **Expected:**
  - URL query becomes tab=notifications and the Notifications tab is active.
  - Header title "Notifications"; subtitle shows "N notification(s)" when there are any, else "Stay updated on your bookings".
  - Notifications render as cards (unread cards have pink background + pink dot + pink title), each with title, relative time, body, and an uppercase type label (e.g. "New Booking", "New Message", "Announcement").
  - Empty state otherwise: "No notifications yet" with helper text.
  - GraphQL operation: myNotifications (getMyNotifications).
- **Source:** src/app/containers/ChatPage/NotificationsPage/index.tsx

### CHAT-11: Mark all notifications as read
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** Logged in with at least one UNREAD notification (so the action button renders).
- **Steps:**
  1. Open the Notifications tab
  2. [MUTATING] Click "Mark all as read"
- **Expected:**
  - All cards lose the unread (pink) styling and dots; the "Mark all as read" button disappears once unread count is 0.
  - The Notifications tab badge count clears/decreases.
  - GraphQL operation: markAllNotificationsAsRead, then refetch.
- **Source:** src/app/containers/ChatPage/NotificationsPage/index.tsx (handleMarkAllAsRead)

### CHAT-12: Clicking a notification marks it read and routes by type
- **Priority:** P1  **Persona:** Guest
- **Preconditions:** Logged in with at least one notification (ideally unread).
- **Steps:**
  1. Open the Notifications tab
  2. [MUTATING] Click an unread notification card
- **Expected:**
  - The notification is marked read (markNotificationAsRead) and loses unread styling.
  - Navigation occurs based on type:
    - ChatMessage -> /chat/<chatId>
    - Broadcast -> /broadcast/<broadcastId>
    - DamageClaim* -> /damage-claims/<id> or the role-appropriate booking page
    - Otherwise with a bookingId -> /cars/host/booking/<id> (host view) or /cars/guest/booking/<id> (guest view)
- **Source:** src/app/containers/ChatPage/NotificationsPage/index.tsx (handleNotificationClick)

### CHAT-13: Load more notifications pagination
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** Logged in with more than 20 notifications (so hasMore is true).
- **Steps:**
  1. Open the Notifications tab
  2. Scroll to the bottom and click "Load more"
- **Expected:**
  - Additional notifications (next page of 20) append to the list; button shows "Loading..." while fetching and disappears when no more pages.
  - GraphQL: myNotifications fetchMore with incremented page.
- **Source:** src/app/containers/ChatPage/NotificationsPage/index.tsx (handleLoadMore)

### CHAT-14: Broadcast/announcement detail page renders
- **Priority:** P2  **Persona:** Guest
- **Preconditions:** A broadcast notification exists (type "Announcement"). Obtain a broadcastId by clicking a Broadcast notification, or navigate directly if an id is known.
- **Steps:**
  1. From the Notifications tab, click an "Announcement" notification (or navigate to https://staging.autopilotee.com/broadcast/<broadcastId>)
- **Expected:**
  - A card renders with a pink gradient header containing the broadcast title and "Posted <Month d, yyyy at h:mm a>".
  - Body shows optional summary and the broadcast HTML content.
  - A "Back to Notifications" button is present and returns to /chat?tab=notifications.
  - If the id is invalid/removed: "Announcement not found" with helper text.
  - GraphQL operation: broadcast (getBroadcast).
- **Source:** src/app/containers/ChatPage/BroadcastDetailPage/index.tsx
