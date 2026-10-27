# Feature Specification: Daily Chat Digest

**Feature Branch**: `001-daily-chat-digest`

**Created**: 2026-10-27

**Status**: Draft

**Input**: User description: "GMR Bot is a Telegram bot that joins a chat and performs both automatic and manual actions. Every day at 00:00 it reviews the chat and publishes a short, well-presented summary of what happened to whom and who did what, with the emphasis on people (for example, 'Misha's friends had a son')."

## Clarifications

### Session 2026-10-27

- Q: What should count as personal news in the daily recap? → A: Notable personal events only (family, health, relationships, moves, jobs, and achievements, and who was involved). Plans, jokes, greetings, and small talk are left out.
- Q: Which messages may supply personal news for the recap? → A: Only text a member typed themselves, including replies. Forwards and media captions are left out.
- Q: Whom should the recap name for being silent, and what should it say when part of the day was not visible? → A: Every fully visible day, name each member who typed no text and state how many consecutive full days that silence has lasted. From the second such day, also say they have been silent for a long time. When only part of the day was visible, publish that part and say in Russian that the rest was not visible. A partly visible day does not add to the full-day count.
- Q: If the bot cannot publish the recap at midnight, what should the chat receive? → A: Publish that day's recap once during the same new Minsk day, as soon as it can be produced. If that new day ends with neither a recap nor a failure note, do not publish that day later.
- Q: If several attempts happen during the same day, what may the chat see after the first publication? → A: At most one recap. If a failure note is already posted and a later attempt produces the recap, post that recap once and add nothing else. A repeat after the recap posts nothing. At most 3 attempts. The `/logs` command publishes every action, successful and unsuccessful.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Read the day's personal news (Priority: P1)

A member opens the chat after midnight and reads one short recap of the previous day. The recap names people and reports notable personal events: family, health, relationships, moves, jobs, and achievements, in the spirit of "Misha's friends had a son." Plans, jokes, greetings, and small talk are left out. It also names each member who typed no text on a fully visible day and states how many full days in a row that silence has lasted. From the second full day, it says that member has been silent for a long time. If only part of the day was visible, the recap says in Russian that the rest was not visible. It is arranged so it can be read in a couple of minutes, and it is written in Russian.

**Why this priority**: This is the daily reason members notice the bot. The recap is useful on its own.

**Independent Test**: Leave a fully visible day that includes a notable personal event a member typed, a notable event that appears only as a forward or a media caption, plus some plans and small talk, in a chat the bot has joined. Include one member who typed nothing. At midnight Minsk time, confirm that the chat receives a Russian recap of the typed notable events only, that the silent member is named with a count of 1 full day, and that a member can retell it without scrolling the day's messages.

**Acceptance Scenarios**:

1. **Given** the chat contained notable personal events during the previous fully visible Minsk calendar day, **When** midnight Minsk time passes, **Then** the chat receives a short Russian recap of those events centered on the people involved, and it names each member who typed no text that day with that member's full-day silence count.
2. **Given** notable personal events about several people, **When** a member reads the recap, **Then** they can tell who each event was about and what happened, without reading the day's messages.
3. **Given** a message from that day is no longer in the chat at midnight, **When** the recap is published, **Then** that message is not reported as personal news.
4. **Given** the previous fully visible day had no notable personal events and every member typed text, **When** midnight passes, **Then** the chat receives no filler recap.
5. **Given** an attempt cannot compose the recap and the chat has no failure note yet, **When** that attempt ends, **Then** the chat receives one clear Russian explanation.
6. **Given** a failure note is already in the chat and a later attempt among the three produces the recap, **When** that attempt ends, **Then** the chat receives that recap once and no second failure note.
7. **Given** that day's recap is already in the chat, **When** another attempt runs, **Then** the chat receives nothing further.
8. **Given** three attempts have failed to produce the recap, **When** the third attempt ends, **Then** the chat has one Russian failure explanation and the bot makes no fourth attempt.
9. **Given** one notable personal event was typed by a member, including in a reply, and another appears only as a forward or a media caption, **When** the recap is published, **Then** the recap includes only the event the member typed.
10. **Given** a fully visible day had no notable personal events and at least one member typed no text, **When** midnight passes, **Then** the chat receives a Russian recap that names each such member, states each full-day silence count, and does not invent personal events.
11. **Given** a member typed no text on a fully visible day and had typed text on the previous fully visible day, **When** that day's recap is published, **Then** it names the member, states 1 full day, and does not say the silence has been long.
12. **Given** a member typed no text on two consecutive fully visible days, **When** the second day's recap is published, **Then** it names the member, states 2 full days, and says the member has been silent for a long time.
13. **Given** the bot could read only part of the day, **When** the recap is published, **Then** it covers that part, says in Russian that the rest of the day was not visible, and does not add that day to anyone's full-day silence count.
14. **Given** the bot could not publish at midnight but can publish later that same new Minsk day, **When** it becomes able to publish, **Then** the chat receives that day's recap once, within 3 attempts.
15. **Given** a new Minsk day ended with neither that day's recap nor a failure note, **When** the next midnight passes, **Then** the missed day is not published.

---

### User Story 2 - Read what the bot did (Priority: P2)

A member sends `/logs` in the chat and receives, in that same chat, a Russian list of what the bot did for this feature: every recorded action, whether it succeeded or failed.

**Why this priority**: Members need a way to see failed attempts as well as published recaps. The daily recap is useful without this list.

**Independent Test**: Cause one successful recap publish and one failed attempt in a chat. Send `/logs` there. Confirm the reply is in that chat, in Russian, lists both the success and the failure, and contains no message text.

**Acceptance Scenarios**:

1. **Given** the bot has both published a recap and failed an attempt in a chat, **When** a member sends `/logs` in that chat, **Then** the bot replies in that chat in Russian with both actions, and the reply contains no message text and no recap body.
2. **Given** actions were recorded in another chat, **When** a member sends `/logs` in this chat, **Then** the reply includes only this chat's actions.
3. **Given** the bot has no recorded actions for this chat, **When** a member sends `/logs`, **Then** the bot replies in Russian that there is nothing recorded yet.
4. **Given** the action list does not fit in one message, **When** a member sends `/logs`, **Then** the bot continues the list in the same chat.
5. **Given** the log cannot be shown, **When** a member sends `/logs`, **Then** the bot replies with a clear Russian explanation and does not stay silent.

---

### Edge Cases

- The previous day is too long for one readable message: the recap continues in the same chat as a short sequence, still centered on people, still in Russian, and still includes the silence counts.
- The bot joined mid-day, or could not read part of the day: the recap covers only the portion it could read and says in Russian that the rest of the day was not visible. Members with no typed text in the visible portion are named for that portion only, without a full-day count. That day does not increase or reset anyone's full-day silence count. If the bot could read nothing because of a failure, members get an explanation instead of a silence list or an empty claim that nothing happened.
- The bot has no earlier mark for a member: the first fully visible day with no typed text is reported as 1 full day. The recap does not invent a longer count.
- A day of plans, jokes, greetings, or small talk, with no notable personal event: the bot posts no event recap. On a fully visible day it still names members who typed no text and states their full-day counts. If every member typed text, it posts nothing.
- A notable personal event that appears only as a forward or as a caption on media: it is left out of the recap. The same event typed by a member, including in a reply, is included. A forward or a caption does not count as that member having typed.
- A join, leave, title change, or other membership notice: it is not treated as personal news. A person who was a member during that Minsk day and typed no text is still named on a fully visible day.
- The same personal event was mentioned many times: the recap tells it once, attached to the people involved.
- The bot cannot publish at midnight: it publishes that day's recap once during the same new Minsk day, as soon as it can, within 3 attempts. Lateness alone is not a failure note. If that new Minsk day ends with neither a recap nor a failure note, that day is not published later. A missed publication does not erase a fully read day's typed/not-typed mark.
- Three attempts fail: the chat has one Russian failure explanation and no fourth attempt. Each failed attempt is still visible through `/logs`.
- `/logs` is sent from outside a chat the bot has joined, or the sender is not a member: the bot does not publish that chat's action list.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: In each chat the bot has joined, the daily recap applies to that chat only. A recap in one chat does not appear in another chat.
- **FR-002**: At midnight Minsk time, the bot publishes a recap of the previous Minsk calendar day in that same chat when that day has a notable personal event, a member who typed no text on a fully visible day, or a part the bot could not read. If that publish is late, the bot publishes that same recap once during the new Minsk day, as soon as it can be produced. If that new day ends with neither a recap nor a failure note, the missed day is not published later. The recap does not repeat earlier days' events. A silence count may refer to earlier full days without retelling those days.
- **FR-003**: The recap is written in Russian and centered on people. Personal news means notable personal events only: family, health, relationships, moves, jobs, and achievements, and who was involved. Plans, jokes, greetings, and small talk are left out. Silence notes are not personal news. Related events about the same person are grouped together. A member can retell the personal news from the recap alone.
- **FR-004**: The recap is short enough to read in a couple of minutes. When one message cannot hold the day, it continues in the same chat without changing that person-centered form. Silence counts stay inside that same recap.
- **FR-005**: The recap includes personal news only from text a member typed themselves that day, including replies, when that text is still in the chat and the bot was able to read it. Forwards and media captions are left out, even when they describe a notable personal event. Only text a member typed themselves counts as that member having written.
- **FR-006**: When a fully visible day has no notable personal events and every member typed text, the bot posts no filler recap. Plans, jokes, greetings, and small talk alone do not count as personal news. A member who typed no text still causes a recap, and that recap does not invent personal events.
- **FR-007**: When an attempt cannot produce the recap during the new Minsk day, and the chat does not already have a failure explanation for that day, the bot posts one clear Russian explanation. It does not stay silent during that day. A later attempt that produces the recap posts that recap once. Lateness alone is not a failure note. The bot posts at most one failure explanation for that day. After the new Minsk day ends, the bot does not post a late recap or a late explanation for that missed day.
- **FR-008**: Membership notices, such as members joining or leaving, are not recapped as personal news.
- **FR-009**: The bot may read a day's messages in order to write that day's recap. It does not keep a lasting copy of message text, and routine records of the recap do not include message text or the recap body. It may keep, per member and per fully visible Minsk day, only whether that member typed any text, so it can state the full-day silence count. It may keep an action record per chat: what it did, which Minsk day the action concerns, whether it succeeded, and the attempt number when there is one.
- **FR-010**: User-facing recaps, failure explanations, and `/logs` replies for this feature are in Russian.
- **FR-011**: On a fully visible day, the recap names each person who was a member of the chat that day and typed no text of their own, including replies. For each such member it states the full-day silence count: the number of consecutive fully visible Minsk days, ending that day, on which the member typed no text. The day being recapped counts as day 1 of a new run when the previous fully visible day included typed text from that member, or when no earlier mark exists. When the count is 2 or more, the recap also says the member has been silent for a long time.
- **FR-012**: When the bot could read only part of the day, the recap covers that part and says in Russian that the rest of the day was not visible. It does not treat that day as a full silent day, does not add it to a full-day count, and does not reset existing counts. Members with no typed text in the visible part are named as having no visible messages, without a full-day count. When the bot could read nothing, it posts the failure explanation and does not name the chat as silent.
- **FR-013**: For each chat and each recap day, the bot makes at most 3 attempts during the new Minsk day. An attempt is one try to produce and publish that day's recap. Publishing the recap ends the attempts; a later try posts nothing. After 3 attempts with no recap, the bot does not try again. The day ending also ends the attempts, even when fewer than 3 were used.
- **FR-014**: A member of a chat the bot has joined can send `/logs` in that chat. The bot replies in that same chat with every recorded action for this feature in that chat, both successful and unsuccessful. The reply does not include another chat's actions, message text, or the recap body. It names the action, the Minsk day it concerns, whether it succeeded, and the attempt number when the action is an attempt. A long list continues in the same chat. If the log cannot be shown, the bot replies with a clear Russian explanation. If nothing is recorded, the reply says so.

### Key Entities

- **Chat**: A Telegram conversation the bot has joined. The recap belongs to one chat and is visible to its members.
- **Member**: A person in the chat during that Minsk day. The bot is not a member for silence notes.
- **Daily recap**: The Russian, person-centered account of notable personal events from one previous Minsk calendar day in one chat, plus silence notes for members who typed no text. Personal news is limited to family, health, relationships, moves, jobs, and achievements, and who was involved.
- **Full-day silence count**: For one member, the number of consecutive fully visible Minsk days ending on the recap day during which that member typed no text. A partly visible day is not part of the count.
- **Action record**: One thing the bot did for this feature in one chat, successful or unsuccessful, without message text or recap body. `/logs` shows these records.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: When the midnight run can publish, members can read the Russian recap in that chat by 00:05 Minsk time.
- **SC-002**: In a review of five sampled days that contained notable personal events in text members typed themselves, including replies, a member who did not read the raw chat can correctly retell at least 90% of those events (family, health, relationships, moves, jobs, and achievements) and who was involved. Events that appear only in forwards or media captions are outside that count.
- **SC-003**: In that same review, every recap can be read in under 2 minutes, and a reviewer can retell it as a short account of people, events, and stated silences.
- **SC-004**: Whenever the recap cannot be produced during the new Minsk day, members see a plain-language Russian explanation in the chat before that day ends. After that day ends, no explanation for it is added.
- **SC-005**: On a fully visible day, every member who typed no text is named, and the stated full-day count equals that member's current run of consecutive fully visible silent days. A count of 2 or more is also described as a long silence. A partly read day does not change any full-day count.
- **SC-006**: When the midnight run is late, that day's recap is published once before the next midnight in Minsk, and within 3 attempts. A day that ends with neither a recap nor a failure note is not published after that next midnight.
- **SC-007**: For one chat and one recap day, the bot attempts publication at most 3 times. After the recap is in the chat, no second recap is posted. After a failure explanation is in the chat, no second explanation is posted. A later success among the remaining attempts still posts the one recap.
- **SC-008**: After a mix of successful and failed actions in one chat, a member who sends `/logs` there can match each reply entry to those actions, including both outcomes. The reply contains no message text and no other chat's actions.

## Assumptions

- The day boundary is midnight in Minsk, because the chat language is Russian and the community context is Belarusian. "Each day at 00:00" means the close of that Minsk calendar day. A full day is a Minsk calendar day the bot could read completely.
- "The whole chat" means notable personal events that members typed themselves that day, including replies, from every member who did so, plus a silence note for every member who typed no such text on a fully visible day. Forwards, media captions, plans, jokes, greetings, and small talk are left out.
- The conversation is a group chat. The user's word "channel" is treated as that shared chat.
- A fully visible day with no notable personal events and no silent member produces no filler recap. Silence notes are not filler. Failure during the new Minsk day is different: members are told before that day ends. After that day ends, a missed recap is not published.
- Messages that are no longer in the chat at midnight are omitted, whoever removed them.
- Reviewing messages against the Republican list of extremist materials is a separate feature. This feature does not perform that review.
- The only manual command this feature adds is `/logs`. Any member of a joined chat can send it there. The reply lists that chat's recorded actions and does not copy message text or the recap body.
