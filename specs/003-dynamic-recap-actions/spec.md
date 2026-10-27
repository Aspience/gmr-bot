# Feature Specification: Dynamic Recap Actions

**Feature Branch**: `003-dynamic-recap-actions`

**Created**: 2026-10-27

**Status**: Draft

**Input**: User description: "определение действий, которые попадут в реап, он же дайджест - должно быть динамическим. Никаких "тегов" вручную перечисленных"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Read what changed for people (Priority: P1)

A member opens the chat after midnight and reads the day's recap. The recap names people and reports what has already happened to them or what they have already done, when that change is something another member would retell. It does this whether or not the happening fits a familiar topic. Plans, jokes, greetings, and small talk stay out. The recap is Russian prose about people. It does not sort the day into topic labels.

**Why this priority**: Members miss the day, not a catalog of topics. A closed list of labels drops real news that simply has no label yet.

**Independent Test**: In a chat the bot has joined, leave one fully visible day where a member types that their dog died, another member types only greetings and small talk, and a third member types a plan for tomorrow. At the usual recap time, confirm the recap includes the dog and who it belonged to, includes neither the greetings nor the plan, and contains no topic label.

**Acceptance Scenarios**:

1. **Given** a member typed that something has already happened to a person, or that a person has already done something that changes that person's situation, **When** the recap is published, **Then** the recap names the people involved and tells that happening in Russian.
2. **Given** that happening is not one of the kinds used in earlier examples, **When** the recap is published, **Then** the happening is still included.
3. **Given** the day also contains plans, jokes, greetings, and small talk, **When** a member reads the recap, **Then** those do not appear, and a completed change told in a joking tone still appears.
4. **Given** the same happening was typed several times, **When** the recap is published, **Then** it is told once, with the people involved.
5. **Given** several completed changes concern the same person, **When** a member reads the recap, **Then** those changes are grouped around that person.
6. **Given** a day of only plans, jokes, greetings, and small talk, **When** the recap time passes, **Then** the recap does not invent a happening.
7. **Given** two members give conflicting accounts of the same change, **When** the recap is published, **Then** each account is attributed to the member who gave it, and the recap adds no third version.
8. **Given** a published recap, **When** a member reads it, **Then** it has no topic label, hashtag, or category heading on any happening.

---

### User Story 2 - Include a new kind of news without a label (Priority: P2)

A kind of personal news shows up that nobody has written down as a topic. Members type it as something that already changed a person's situation. The recap includes it. Nobody adds a label first, and members are not asked to mark the message.

**Why this priority**: The open rule is what keeps the recap from going stale. Story 1 is still useful for happenings the community already knows how to recognize.

**Independent Test**: Choose a completed personal change that appears in no written topic list and in none of the examples for this feature. Have a member type it. Publish the recap. Confirm the change and the people are in the recap, and confirm that no topic list was edited and no member was asked to apply a label.

**Acceptance Scenarios**:

1. **Given** a member types a completed change in a person's situation of a kind that has no topic label anywhere in the product, **When** the recap is published, **Then** the recap includes that change and who was involved.
2. **Given** that kind of change has never appeared before, **When** it is included, **Then** no person has edited a list of topics, and no member has been asked to tag the message.
3. **Given** the same kind of completed change is typed on a later day, **When** that later recap is published, **Then** it is included again under the same rule.
4. **Given** a member types only that they might do that thing later, **When** the recap is published, **Then** the plan is left out.

---

### Edge Cases

- A happening is told as a joke ("haha, I got fired"): the firing is included, because the situation already changed. The joke tone is not retold as the news.
- A message names no person ("it rained all day"): it is left out. The same weather is included when it is a change for a person ("Misha's apartment was flooded").
- Ordinary completed activity that does not change a person's situation ("I had soup", "I watched a show") is left out.
- A plan that sounds important ("I'm moving next year", "I might get a dog") is left out until a member reports that it has happened.
- A message describes a change but does not say who it happened to: it is left out rather than assigned to a guessed person.
- Two members describe the same change differently: both accounts are attributed to the people who wrote them. The recap does not declare a winner and does not add facts from neither account.
- Many different changes occur in one day: each distinct change is told once, grouped by person, in a recap a member can still read in a couple of minutes. A long day continues in the same chat, as the daily recap already does.
- A completed change appears only as a forward or as a media caption: it stays out of the recap. The daily recap's source rule is unchanged.
- A membership notice, such as a join or a leave, is not treated as one of these happenings.
- Nothing in the day is a completed change in a person's situation: the recap does not invent one. Silence notes, when the daily recap already requires them, are not happenings and are not given topic labels.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: For each chat and each recap day, the bot decides which happenings enter the recap from that day's own messages. The decision does not use a fixed or editable list of topic labels.
- **FR-002**: A happening enters the recap when text a member typed themselves reports that something has already happened to a person, or that a person has already done something, and that this changes that person's situation. The recap names who was involved.
- **FR-003**: A happening of a kind that has never been given a topic label still enters when it meets FR-002. No person adds a label first. Members are not asked to tag, classify, or mark messages for the recap.
- **FR-004**: Plans, jokes, greetings, and small talk stay out of the happening narrative. A completed change told in a joking tone still enters. A plan enters only after a member reports that it has happened.
- **FR-005**: The same completed change is treated the same way on another day. Chatter is left out on every day. The open set of happenings does not mean the rule changes from day to day.
- **FR-006**: Each distinct happening is told once. Happenings about the same person are grouped together. The recap does not invent people, happenings, or a version that nobody wrote. Conflicting accounts are attributed to the members who wrote them.
- **FR-007**: The published recap is Russian prose centered on people. It does not attach a topic label, hashtag, or category heading to a happening.
- **FR-008**: When no happening meets FR-002, the recap does not invent one.
- **FR-009**: This feature adds no standing catalog of topics about people, and it does not keep a lasting copy of message text.
- **FR-010**: Silence notes, the Minsk midnight schedule, late publication, the three attempts, failure notes, `/logs`, and which messages may supply text stay as specified for the daily chat digest. Forwards, media captions, and membership notices still do not supply happenings.

### Key Entities

- **Happening**: A completed change in a person's situation, as told in a member's own text: something that has already happened to that person, or something that person has already done. It is not a topic label.
- **Recap narrative**: The Russian, person-centered account of one day's happenings in one chat. It carries no topic labels. Silence notes are separate and are not happenings.
- **Topic label**: A hand-written name for a kind of news. This feature has no list of them, and members do not apply them.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In a review of 8 sampled days, each built around a different completed personal change, and none of those kinds named on a topic list, a member who did not read the raw chat can retell at least 7 of the 8 changes and who was involved.
- **SC-002**: In a review of 8 sampled days that contain only plans, jokes, greetings, and small talk, none of those days produce a happening narrative.
- **SC-003**: In a review of 5 mixed days, every completed personal change is present in the recap and every plan, joke, greeting, and stretch of small talk is absent.
- **SC-004**: Including a completed change whose kind appears on no topic list requires no edit to a list of topics and no tagging by a member. A later day with that same kind of change includes it again.
- **SC-005**: In the samples above, every recap can be read in under 2 minutes and contains no topic label, hashtag, or category heading.
- **SC-006**: In a review of five sampled days that contain a completed change already familiar to the community, such as a friend's child being born, a member who did not read the raw chat can still retell at least 90% of those changes and who was involved.

## Assumptions

- "Dynamic" means the day's messages are judged one by one against a stable, open rule. The rule is the same in every chat. It is not a catalog anyone edits, and it does not grow a private topic list for one chat.
- A change in a person's situation is news another member would retell after missing the day: a loss, a gain, a legal or practical outcome, harm, a death, a new responsibility, or a completed life event. Eating, watching something, greeting people, and joking are not such a change.
- The daily chat digest already publishes the Russian recap, names silent members, and limits sources to text a member typed. This feature replaces only that digest's closed set of topic labels. It does not reopen schedule, attempts, silence counts, `/logs`, privacy of message text, or the exclusion of forwards and captions.
- Familiar news from that digest, such as a friend's child being born, still qualifies when a member reports it as something that has already happened. Those examples are not an exclusive list.
- Members do not get a new command to include, exclude, or label topics.
- Conflicting accounts are both worth retelling as what those members said. The bot does not run a separate dispute process.
