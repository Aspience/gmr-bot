# Feature Specification: Listed Source Review

**Feature Branch**: `002-listed-source-review`

**Created**: 2026-10-27

**Status**: Draft

**Input**: User description: "Every message is checked against the Republican list of extremist materials at https://mininform.gov.by/ru/extrim-material-ru. The list can be refreshed by a manual command. If a message matches, the bot posts a notice with Delete and Allow buttons, visible to every member. If anyone chooses Delete, both the notice and the matched message disappear."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Decide whether a listed source stays (Priority: P1)

A member sends a message that points at, or is forwarded from, an information product on the Republican list of extremist materials. Every member sees one notice naming the listed source and offering «Удалить» and «Допустить». The original message stays until someone chooses. One member's choice applies to the whole chat.

**Why this priority**: Members need a shared, explicit decision before a listed source remains or is removed. The review is useful on its own.

**Independent Test**: Post a message that links to a source on the loaded list, then have one member choose «Удалить» and, in a second case, «Допустить». Confirm the whole chat sees the same result, and that a message which only mentions a person's name produces no notice.

**Acceptance Scenarios**:

1. **Given** the list is loaded and a member sends a message that links to a listed source or is forwarded from one, **When** the message appears, **Then** every member sees one Russian notice naming that source, with «Удалить» and «Допустить», and the original message is still in the chat.
2. **Given** that notice is open, **When** any member chooses «Удалить», **Then** both the notice and the original message disappear for every member.
3. **Given** that notice is open, **When** any member chooses «Допустить», **Then** the notice disappears for every member and the original message remains.
4. **Given** a member already completed «Допустить» or «Удалить», **When** another member chooses the other action, **Then** the first choice stands for the whole chat.
5. **Given** a message only mentions a person's name and contains no listed address, account, channel, or handle, and is not forwarded from a listed source, **When** it appears, **Then** no notice is posted and the message stays.
6. **Given** a member is not allowed to have the matched message removed, **When** they choose «Удалить», **Then** the original message stays, the notice stays, and the chat is told in Russian why it could not be removed.
7. **Given** a message was allowed, **When** that same message is edited so that it newly references a listed source, **Then** a new notice is shown for the edited message.
8. **Given** a review decision has been recorded, **When** that decision is examined later, **Then** it shows the choice, the listed source, and when the choice was made, and it does not include the text of the message.

---

### User Story 2 - Refresh the official list (Priority: P2)

The official list changes. Any member of the chat can ask the bot to load it again. The bot confirms success or explains failure. Later messages are judged with the list that loaded successfully. Older decisions are left as they are.

**Why this priority**: Source review depends on a current list, and members need a manual way to update it. A previously loaded list still lets review continue while this story is added.

**Independent Test**: Run the refresh command once when the official page is available and once when it is not. Confirm the reply, then post a message and confirm it is judged with the last list that loaded successfully. Run the command again immediately and confirm a polite refusal.

**Acceptance Scenarios**:

1. **Given** a member requests a refresh and the official list loads, **When** the request finishes, **Then** that member sees a Russian confirmation, and a message sent afterward is judged with the updated list.
2. **Given** a list was loaded before and a refresh cannot reach the official list, **When** a member requests a refresh, **Then** they are told the refresh failed and review continues with the previous list.
3. **Given** no list has ever loaded and the official list cannot be reached, **When** a message arrives, **Then** nothing is removed and the chat is told in Russian that source review is unavailable.
4. **Given** a refresh for that chat just succeeded, **When** a member requests another refresh within 5 minutes, **Then** the bot refuses politely in Russian and does not post a second update.
5. **Given** a message was already allowed or removed, **When** the list is refreshed, **Then** that past decision is not reopened and older chat history is not reviewed again.

---

### Edge Cases

- Two members choose different actions before either choice has been applied: «Удалить» is applied, and «Допустить» is not.
- The matched message is already gone when a member chooses: the notice disappears, and the chat is not left with a stale prompt.
- A message references more than one listed source: one notice names each matched source and still offers a single «Удалить» and a single «Допустить».
- The same unchanged message matches again: a second notice is not added while the first notice is open, and an allowed message is not reviewed again until its content changes.
- A message only links to the Ministry of Information page that publishes the list: that page address alone is not a match.
- A join, leave, title change, or other membership notice: it is not source-reviewed.
- A link or listed handle appears in a caption, attachment, or reply, or the message is forwarded from a listed source: it is reviewed the same way as a link in the message text.
- Many messages arrive at once: each matching message still receives its own notice, later messages are still reviewed, and a check that cannot finish is explained in the chat.
- The refresh command is requested from outside the chat, or a notice is acted on by someone who is not a member: the bot does not change that chat.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: In each chat the bot has joined, source review applies to that chat only. A decision in one chat does not affect another chat.
- **FR-002**: The bot judges each new or edited member message against the Republican list of extremist materials published at https://mininform.gov.by/ru/extrim-material-ru, using the last list that loaded successfully for that chat.
- **FR-003**: A message matches only when it references a listed information product by an address, account, channel, handle, or other identifier published for that entry, or when it is forwarded from such a product. A person's name alone does not match. The address of the Ministry page that publishes the list does not match by itself.
- **FR-004**: The bot never removes a member's message on its own. A match produces one review notice and leaves the original message in place until a member chooses «Удалить».
- **FR-005**: The notice is visible to every member of that chat, is written in Russian, names each matched source, and offers exactly two actions, labeled «Удалить» and «Допустить».
- **FR-006**: Only one open notice exists for a given message. An unchanged message that was allowed is not noticed again. A later edit that newly matches produces a new notice.
- **FR-007**: If any member chooses «Удалить», both the notice and the matched message disappear for every member of the chat.
- **FR-008**: If any member chooses «Допустить», the notice disappears for every member and the matched message remains. That choice applies to the whole chat.
- **FR-009**: The first completed choice on a notice applies to the whole chat and is not reversed by a later choice. If «Удалить» and «Допустить» are both accepted before either has been applied, «Удалить» is the choice that applies.
- **FR-010**: If the matched message cannot be removed, or is already gone, the chat receives a clear Russian explanation. A stale notice does not remain after the original message is already gone.
- **FR-011**: Any member of the chat can request a manual refresh of the official list. The requester receives a Russian confirmation or a Russian explanation of failure.
- **FR-012**: Before the first review in a chat, the bot attempts to load the official list. After a successful load, further updates happen when a member refreshes it. A failed refresh keeps the last successful list. If no list has ever loaded, the bot explains that source review is unavailable and does not treat messages as matches or remove them.
- **FR-013**: A refresh does not reopen decisions already made and does not review older chat history.
- **FR-014**: A second refresh request in the same chat within 5 minutes of a successful refresh is refused with a short Russian reply.
- **FR-015**: A burst of messages does not stop later messages from being reviewed. A review that cannot finish is explained in the chat rather than dropped in silence.
- **FR-016**: Membership notices, such as members joining or leaving, are not source-reviewed.
- **FR-017**: The bot remembers, for each decision, the choice, the listed source, which message it applied to, and when it was made, plus when the list was last loaded. It does not keep a lasting copy of message text, and routine records of these actions do not include message text.
- **FR-018**: Only members of the affected chat can refresh the list or choose «Удалить» or «Допустить» for that chat. Other input on these actions is refused with a short Russian explanation.
- **FR-019**: User-facing notices, confirmations, and failure explanations for this feature are in Russian.

### Key Entities

- **Chat**: A Telegram conversation the bot has joined. Notices and decisions belong to one chat and are visible to its members.
- **Official list entry**: One information product on the Republican list of extremist materials, recognized by the addresses, accounts, channels, handles, and other identifiers published for it.
- **Loaded list**: The official list as it was last read successfully for a chat, and when that reading happened.
- **Review notice**: The single chat message that names the matched sources and offers «Удалить» and «Допустить».
- **Member decision**: One chat-wide choice, Delete or Allow, for one message, including which listed sources matched and when the choice was made. It does not include the message text.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Under ordinary conditions, a message that references a listed source receives one notice visible to every member within 1 minute, and the original message is still present until a member chooses.
- **SC-002**: In every observed «Удалить» choice, both the notice and the matched message are absent for every member afterward. In every observed «Допустить» choice, the notice is absent and the matched message remains for every member.
- **SC-003**: Across all matched messages, none are removed unless a member chose «Удалить».
- **SC-004**: A member who requests a list refresh receives a success or failure reply within 1 minute. A message sent after a successful refresh is judged with the refreshed list.
- **SC-005**: Whenever a removal cannot be completed, or source review cannot run because no list is available, members see a plain-language Russian explanation in the chat.

## Assumptions

- The conversation is a group chat whose members can all see messages and press the notice actions. The user's word "channel" is treated as that shared chat.
- «Допустить» removes the notice for everyone and leaves the original message in place. The choice is final for that unchanged message.
- One shared notice in the chat satisfies "shown to each user." Separate private copies are not sent.
- A match follows the identifiers published on the official list (addresses, accounts, channels, handles, and forwards). Ordinary mention of a name is not a match.
- Any member of the chat may refresh the list. A repeat within 5 minutes is refused so one member cannot crowd the chat. The first load is automatic so review can start before anyone discovers the command.
- Members are always told when the refresh, the removal, or the review cannot be completed.
- The bot may read a message in order to judge its source. It does not keep a lasting archive of what people wrote.
- The daily person-centered recap is a separate feature. This feature does not publish that recap. A message left in the chat by «Допустить» stays in the chat.
- This feature does not change the official list, does not review private direct conversations, does not review chats the bot has not joined, and does not add manual actions other than the list refresh and the two notice choices.
