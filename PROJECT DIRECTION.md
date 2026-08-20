# Master Engineering Prompt: DeskcommCRM + WasenderAPI + Visual Workflow + Multi-Variant Experimentation

## Mission

You are an autonomous senior software architect, full-stack engineer, DevOps engineer, QA engineer, security engineer, and Playwright/UI automation engineer.

Your mission is to extend the existing **DeskcommCRM** codebase into a production-grade WhatsApp CRM and experimentation platform while preserving as much of the existing working architecture as possible.

Repository:

`https://github.com/melgarafael/DeskcommCRM`

Automation reference:

`https://github.com/ArnasDon/wacrm` and `https://wacrm.tech/llms.txt`

WasenderAPI documentation:

`https://wasenderapi.com/llms.txt`

The existing DeskcommCRM architecture already contains a substantial foundation: Next.js/React, Supabase Postgres/RLS/Auth/Storage/Realtime, an event-log/worker architecture, CRM/inbox/pipelines, WAHA integration, tests, and server-side API routes. The repository explicitly identifies Supabase Storage as the media layer and `event_log` + workers as the queue/processing mechanism. ([GitHub](https://github.com/melgarafael/DeskcommCRM?utm_source=chatgpt.com))

**Do not rebuild these foundations unnecessarily.**

The objective is not to create a new CRM.

The objective is to make the smallest justified architectural changes required to create the following system:

```text
Customer WhatsApp
        ↓
    WasenderAPI
        ↓
Wasender Adapter
        ↓
Canonical Deskcomm Events
        ↓
Deskcomm CRM
        ↓
Keyword / Workflow Engine
        ↓
Experiment / Variant Engine
        ↓
Existing CRM Workers / Scheduling
        ↓
Wasender Adapter
        ↓
WasenderAPI
        ↓
Customer WhatsApp
```

---

# 1. NON-NEGOTIABLE ENGINEERING PRINCIPLE

Do NOT interpret "preserve the core" as "never modify existing code."

Instead:

> Preserve existing behavior and architecture wherever possible. Modify existing code only when the new requirements genuinely require it. Prefer adapters, extension points, new modules, existing events, existing workers, existing CRM entities, and existing storage over duplicated infrastructure.

Every modification must answer:

1. What existing component is being changed?
2. Why must it change?
3. Why can't the requirement be implemented beside it?
4. What existing behavior depends on it?
5. What regression tests protect that behavior?
6. Can the change be isolated behind an interface?
7. Can the change later be removed or replaced without rewriting the rest of Deskcomm?

Do not perform broad refactors merely for code cleanliness.

Do not rewrite working systems simply because a different architecture appears more elegant.

Do not copy another CRM wholesale into Deskcomm.

---

# 2. EXISTING SYSTEM MUST BE AUDITED FIRST

Before writing implementation code, inspect the repository deeply.

Map at minimum:

```text
WAHA integration
    ↓
WAHA client
    ↓
WAHA webhooks
    ↓
message normalization
    ↓
contacts
    ↓
conversations
    ↓
messages
    ↓
event_log
    ↓
workers
    ↓
automation
    ↓
outgoing messages
    ↓
WAHA
```

Also map:

```text
Supabase schema
RLS
Storage
workers
cron
event_log
rate limiting
idempotency
inbox
pipelines
follow-ups
analytics
authentication
multi-tenancy
existing automation UI
existing tests
Docker
deployment
environment variables
health checks
```

Read the relevant source files, not only README files.

Produce an internal architecture report before implementation.

Do not begin feature implementation until you understand the actual source-code path.

---

# 3. EXISTING DESKCOMMERCRM FOUNDATION

The current DeskcommCRM stack includes:

- Next.js
- React
- TypeScript
- Tailwind/shadcn
- Supabase Postgres
- Supabase Auth
- Supabase Storage
- Supabase Realtime
- RLS
- event_log
- workers
- Upstash Redis
- WAHA
- AI SDK
- existing CRM/inbox/pipeline functionality

The official setup documentation states that Supabase provides the database, authentication and storage, and that the private `whatsapp-media` bucket is used for media through signed URLs. It also states that AI integrations can be omitted for the minimum application startup, while WAHA is required for WhatsApp functionality. ([GitHub](https://github.com/melgarafael/DeskcommCRM/blob/main/docs/SETUP.md?utm_source=chatgpt.com))

Therefore:

### Reuse:

- Supabase
- Postgres
- RLS
- Storage
- Realtime
- event_log
- workers
- existing CRM
- existing conversations
- existing inbox
- existing pipeline
- existing follow-up mechanisms
- existing authentication
- existing observability
- existing rate limiting
- existing tests

Do not introduce another database.

Do not introduce another queue unless the existing worker architecture demonstrably cannot satisfy a requirement.

Do not introduce Cloudflare R2 in V1 unless the repository audit proves that Supabase Storage cannot satisfy the requirement.

---

# 4. AI REQUIREMENT

Do NOT remove the existing AI architecture.

Do NOT replace the existing AI layer.

Do NOT require an AI API key for the initial deployment unless the application demonstrably cannot start without one.

The requirement is:

> AI is OPTIONAL for V1.

The user is deliberately not providing an AI API key initially because automated AI responses are not part of the initial product.

The CRM, WhatsApp integration, workflows, keyword automation, experiments, analytics and human-operated inbox must work without a configured AI provider wherever the existing application architecture allows it.

Do not delete AI-related code.

Do not hardcode another AI provider.

Do not replace the existing AI architecture.

Do not make workflow execution dependent on AI.

Later, an AI provider may be configured without architectural redesign.

---

# 5. CHANGE #1 — REPLACE WAHA WITH WASENDERAPI

The WhatsApp transport layer must be replaced from WAHA to WasenderAPI.

Do not simply rename environment variables.

Perform a complete WAHA → Wasender compatibility analysis.

Create a matrix:

| Deskcomm capability | WAHA implementation | Wasender implementation | Required adapter |
|---|---|---|---|
| session management | inspect | inspect | yes/no |
| connection status | inspect | inspect | yes/no |
| QR connection | inspect | inspect | yes/no |
| inbound webhook | inspect | inspect | yes/no |
| outbound text | inspect | inspect | yes/no |
| image | inspect | inspect | yes/no |
| video | inspect | inspect | yes/no |
| audio | inspect | inspect | yes/no |
| document | inspect | inspect | yes/no |
| message ID | inspect | inspect | yes/no |
| delivery status | inspect | inspect | yes/no |
| contact identity | inspect | inspect | yes/no |
| quoted message | inspect | inspect | yes/no |
| media download | inspect | inspect | yes/no |
| message logs | inspect | inspect | yes/no |
| session status | inspect | inspect | yes/no |
| webhook verification | inspect | inspect | yes/no |

Wasender currently exposes session management, QR-code functionality, message sending, message information, media upload, and multiple webhook events. ([WASenderApi](https://api.wasenderapi.com/api-docs/messages?utm_source=chatgpt.com))

---

# 6. WASENDER AUTHENTICATION — CRITICAL

Never confuse these two credentials:

## Personal Access Token

Account-level credential.

Use only server-side.

It can be used for account/session-management operations supported by WasenderAPI.

Current documentation shows that retrieving a WhatsApp session QR code requires an access token/PAT. ([WASenderApi](https://www.wasenderapi.com/api-docs/sessions/get-whatsapp-session-qr-code?utm_source=chatgpt.com))

## Session API Key

Session-scoped credential.

Current documentation states that the API key is generated after a WhatsApp session is connected and is tied to that specific session. ([WASenderApi](https://www.wasenderapi.com/api-docs/authentication/how-to-authenticate-api-requests-using-bearer-tokens?utm_source=chatgpt.com))

Therefore:

```text
Wasender Account
    │
    └── Personal Access Token
            │
            ├── Session A
            │     └── Session API Key A
            │
            ├── Session B
            │     └── Session API Key B
            │
            └── Session C
                  └── Session API Key C
```

Never substitute one for the other.

Before implementing any endpoint, inspect Wasender documentation and explicitly determine:

- required credential
- credential scope
- required headers
- response structure
- error behavior

Never guess.

Never put the Personal Access Token in frontend/client-side code.

Never expose secrets through `NEXT_PUBLIC_*`.

Never log tokens.

Never commit tokens.

---

# 7. QR-CODE ARCHITECTURE

Do not blindly reproduce WAHA's QR architecture.

Investigate two supported approaches:

### Approach A

Connect/manage the WhatsApp session through Wasender's dashboard.

### Approach B

Deskcomm uses the Wasender Personal Access Token server-side to manage sessions and retrieve QR information.

Wasender currently documents:

```text
POST /api/whatsapp-sessions
GET  /api/whatsapp-sessions/{id}/qrcode
```

and the QR endpoint requires an access token. ([WASenderApi](https://www.wasenderapi.com/api-docs/sessions/get-whatsapp-session-qr-code?utm_source=chatgpt.com))

For V1, choose the approach that requires the least custom code and lowest failure surface.

Do not implement a custom QR/session-management UI merely because WAHA had one.

If dashboard-based connection is sufficient for V1, support session registration/configuration in Deskcomm without recreating the entire Wasender dashboard.

A future V2/V3 may add native session creation/QR management if valuable.

---

# 8. WASENDER WEBHOOK ADAPTER

Create a provider adapter/normalization layer.

Do not allow Wasender-specific payloads to leak throughout the CRM.

Conceptually:

```text
Wasender Webhook
       ↓
WasenderWebhookParser
       ↓
CanonicalWhatsAppEvent
       ↓
Deskcomm
```

The canonical event should contain only the information Deskcomm actually needs, such as:

```text
provider
session_id
provider_message_id
contact identity
sender
recipient
conversation identity
message type
text
media metadata
timestamp
quoted message
direction
raw event reference
```

Wasender provides message-received/upsert events and includes identifiers such as message ID, remote JID, sender phone information and message body. ([WASenderApi](https://wasenderapi.com/api-docs/webhooks/webhook-message-upsert?utm_source=chatgpt.com))

Support webhook signature verification.

Wasender documents `X-Webhook-Signature` verification and recommends returning HTTP 200 quickly while processing asynchronously. ([WASenderApi](https://wasenderapi.com/api-docs/webhooks/webhook-setup?utm_source=chatgpt.com))

Implement:

```text
verify
↓
acknowledge
↓
persist/idempotency check
↓
process asynchronously
```

Never perform long-running workflow execution inside the webhook request itself.

---

# 9. WEBHOOK IDEMPOTENCY

This is mandatory.

If Wasender sends the same event twice:

```text
event A
event A
```

the system must process it exactly once.

Test:

```text
same provider message ID
same event
multiple deliveries
concurrent deliveries
```

Expected:

```text
1 CRM message
1 workflow trigger
1 experiment assignment
0 duplicate outgoing messages
```

Use the existing event/idempotency infrastructure if possible.

Do not invent a second idempotency system unless required.

---

# 10. CHANGE #2 — KEYWORD-BASED WORKFLOW ENGINE

The new automation system must support:

### Triggers

At minimum:

```text
keyword
incoming WhatsApp message
new contact
contact/tag event
pipeline stage
webhook
schedule
```

The first V1 trigger must be:

```text
incoming WhatsApp message → keyword
```

The engine should normalize incoming Wasender events before trigger matching.

Example:

```text
Wasender message
       ↓
Deskcomm event
       ↓
Keyword matcher
       ↓
Workflow
```

---

# 11. VISUAL WORKFLOW BUILDER

Deskcomm already has automation functionality, but the current architecture should be inspected to determine whether its UI is sufficient for a true visual workflow editor.

Do not discard existing automation.

Use it where possible.

However, the target UX should support a visual workflow builder inspired by the useful concepts of WA CRM.

WA CRM currently documents a no-code automation editor with triggers and steps, and separates Automations from more complex Flows. ([wacrm](https://wacrm.tech/docs/automations?utm_source=chatgpt.com))

Use WA CRM as an architectural/UX reference, not as a second application to merge into Deskcomm.

Do not copy its entire codebase.

---

# 12. WORKFLOW NODE TYPES

V1 should contain only the minimum useful node set:

### Trigger

```text
Keyword
```

### Actions

```text
Send text
Send image
Send audio
Send video
Send document
```

### Control

```text
Wait
Condition
Stop
```

V2 may add:

```text
Random wait
Tag
Remove tag
Move pipeline stage
Assign agent
Webhook
Update contact
```

V3 may add more advanced branching and reusable workflow components.

---

# 13. RANDOM DELAYS

Support:

```text
minimum delay
maximum delay
```

Example:

```text
min = 60 seconds
max = 180 seconds
```

The production executor chooses a value within the range.

The testing environment must support deterministic random seeds so tests are reproducible.

Never make automated testing depend on real-time random delays.

---

# 14. WORKFLOW PERSISTENCE

Every workflow execution must have durable state.

Example:

```text
workflow
workflow_execution
workflow_step
workflow_execution_state
```

Reuse existing event/worker infrastructure wherever possible.

A workflow waiting for 30 minutes must survive:

- worker restart
- application restart
- deployment
- temporary provider outage

It must not restart from step 1.

It must not send duplicate messages.

WA CRM's current implementation is a useful reference here: its Wait step persists pending executions and a cron process drains them. ([wacrm](https://wacrm.tech/docs/automations-and-cron?utm_source=chatgpt.com))

Do not copy its implementation blindly; adapt the concept to Deskcomm's existing workers/event architecture.

---

# 15. CHANGE #3 — MULTI-VARIANT EXPERIMENTATION ENGINE

This is a new custom subsystem.

It must support an arbitrary number of variants.

Do NOT hardcode:

```text A/B
```

Use:

```text Variant 1
Variant 2
Variant 3
...
Variant N
```

210 variants must be supported without schema redesign.

---

# 16. VARIANT MODEL

Conceptually:

```text
Campaign
    │
    ├── Variant
    │
    ├── Variant
    │
    └── Variant
```

Each variant needs:

```text
id
campaign_id
name
status
traffic_weight
workflow_id
created_at
updated_at
```

A variant must be:

```text
active
inactive
```

The user must be able to:

- activate
- deactivate
- reactivate
- edit
- duplicate
- inspect
- link to a workflow

---

# 17. VARIANT ASSIGNMENT

When a qualifying customer triggers a campaign:

```text
Customer
    ↓
Campaign
    ↓
Variant allocator
    ↓
Variant N
```

Once assigned:

```text
customer + campaign → variant
```

must persist.

The same customer must not randomly move from Variant A to Variant B during the same experiment unless the user explicitly resets/reassigns the experiment.

---

# 18. VARIANT STATISTICS

Each variant must expose statistics including at minimum:

```text
assigned contacts
messages sent
messages delivered if available
replies
reply rate
workflow starts
workflow completions
workflow stops
conversions
conversion rate
average time to reply
```

Where provider events support the metric, use actual Wasender status/events.

Wasender documents message sent events and message information/status functionality. ([WASenderApi](https://wasenderapi.com/api-docs/webhooks/webhook-message-sent?utm_source=chatgpt.com))

Do not invent delivery data when the provider did not confirm it.

---

# 19. REPLY ATTRIBUTION

This is mandatory.

Example:

```text
Customer 123
Campaign PROMO
Variant B
Workflow B
Message 55
```

Customer replies.

The system must know:

```text
reply → Customer 123
reply → Campaign PROMO
reply → Variant B
reply → Workflow B
reply → relevant stage
```

This attribution must survive workflow progression.

---

# 20. VARIANT-SPECIFIC WORKFLOWS

Each variant must be capable of linking to its own workflow.

Example:

```text
Variant A → Workflow A
Variant B → Workflow B
Variant C → Workflow C
```

They may share a workflow template, but the system must permit completely different workflows.

Example:

```text
Variant A:
Text → Image → Wait → Audio

Variant B:
Text → Video → Wait → Text

Variant C:
Audio → Image → Condition → Follow-up
```

---

# 21. VARIANT DEACTIVATION

If Variant B is deactivated:

```text
Existing customers already assigned to B:
continue according to policy.

New customers:
must not receive B.
```

Do not silently delete variant data.

Historical statistics must remain intact.

---

# 22. CONTENT AND MEDIA

Deskcomm already uses Supabase Storage for media and has a private `whatsapp-media` bucket. ([GitHub](https://github.com/melgarafael/DeskcommCRM/blob/main/docs/SETUP.md?utm_source=chatgpt.com))

Do not add Cloudflare R2 in V1 unless testing demonstrates a real limitation.

Create a clean storage abstraction if one does not already exist.

Future architecture:

```text
StorageProvider
    │
    ├── SupabaseStorageProvider
    │
    └── future R2StorageProvider
```

For V1:

```text
Supabase Storage
```

is the only implementation.

---

# 23. MEDIA WORKFLOW NODES

Support:

```text
text
image
audio
video
document
```

Wasender currently supports media upload and sending for image, video, audio and document types. Its upload endpoint accepts binary or Base64 and returns a temporary public URL. ([WASenderApi](https://api.wasenderapi.com/api-docs/messages/upload-media-file?utm_source=chatgpt.com))

The media pipeline should be:

```text
Workflow Builder
       ↓
Supabase Storage
       ↓
Media resolver
       ↓
Wasender adapter
       ↓
Wasender media upload if required
       ↓
Wasender send-message
```

Do not expose private Supabase credentials.

Do not assume Wasender can access a private Supabase URL.

Explicitly test:

- signed URL
- media upload
- MIME type
- file size
- temporary URL expiration
- send success
- failure recovery

---

# 24. PROGRAMMATIC WORKFLOW CREATION

This is a first-class requirement.

The system must support workflow creation without using the visual builder.

Example intent:

> "For this audio file, this text message and this image, create the sequence and add it to workflow Variant B."

The coding agent or external automation tool should be able to create the same workflow representation that the UI creates.

Do NOT create a separate "programmatic workflow format" that behaves differently from the UI.

The UI and API must use the same underlying workflow model.

Architecture:

```text
                 Workflow Definition
                  /              \
                 /                \
        Visual Builder          API
              \                  /
               \                /
                Workflow Engine
```

This is critical for scaling to 210+ variants.

---

# 25. PROGRAMMATIC API

Create a documented server-side/API interface for:

```text
create campaign
create variant
activate/deactivate variant
create workflow
update workflow
duplicate workflow
attach media
attach text
attach delay
attach condition
link variant → workflow
bulk import variants
bulk import workflows
```

The exact endpoints must follow Deskcomm's existing API conventions.

Do not invent an unrelated API architecture.

---

# 26. BULK VARIANT CREATION

Support importing many variants programmatically.

Example:

```text
campaign.json
variants.json
media/
messages/
```

or another format chosen after repository analysis.

The importer must validate before writing.

If 210 variants are imported:

```text
all 210 valid
```

must produce:

```text
210 variants
210 valid assignments/workflow links
```

If variant 173 is invalid:

```text
do not silently create partial corrupted state
```

Use transactional or staged import behavior where appropriate.

---

# 27. PERSONALIZATION WITHOUT AI

The system must support deterministic variables such as:

```text
{{first_name}}
{{last_name}}
{{phone}}
{{stage}}
{{company}}
{{product}}
```

The exact available variables must follow existing Deskcomm data models.

Do not require an AI API.

AI-generated content may be imported later through Claude Code/OpenCode or another external process.

---

# 28. TESTING ARCHITECTURE

Testing is not an afterthought.

Every implementation phase must have tests before proceeding.

Use multiple layers:

```text
Unit
↓
Property-based
↓
Database integration
↓
Contract
↓
Provider simulation
↓
Workflow/state-machine
↓
Concurrency/idempotency
↓
UI E2E
↓
Real Wasender
↓
Real WhatsApp
```

---

# 29. FAKE WASENDER PROVIDER

Before connecting a real WhatsApp number, build a test provider.

It must simulate:

```text
receive message
send text
send image
send audio
send video
send document
message sent
message delivered
message read if available
message failed
duplicate webhook
session event
```

The application should interact with it through the same provider abstraction used by the real Wasender integration.

This lets us test:

```text
Customer
 ↓
Fake Wasender
 ↓
Deskcomm
 ↓
Workflow
 ↓
Fake Wasender
```

without sending any real WhatsApp message.

---

# 30. WORKFLOW STATE-MACHINE TESTING

Every workflow must be testable as a state machine.

Example:

```text
TRIGGERED
    ↓
STEP_1
    ↓
WAITING
    ↓
STEP_2
    ↓
WAITING
    ↓
REPLY_RECEIVED
    ↓
BRANCH
    ↓
COMPLETED
```

Test:

- normal execution
- reply
- no reply
- timeout
- cancellation
- deactivation
- worker restart
- duplicate event
- provider failure
- invalid media
- missing contact
- missing workflow
- concurrent event

---

# 31. PROPERTY-BASED TESTING

Generate large numbers of cases.

Test invariants such as:

```text
one incoming message cannot create duplicate workflow execution

one customer cannot have two active assignments
for the same campaign

inactive variants cannot receive new assignments

completed workflows cannot execute another step

a workflow cannot send to the wrong contact

a variant reply cannot be attributed to another variant

a session cannot send through another session

tenant A cannot access tenant B data
```

Run these tests repeatedly.

---

# 32. CONCURRENCY TESTING

Simulate:

```text
10 customers
100 customers
1,000 virtual customers
```

sending messages simultaneously.

Verify:

```text
correct contact
correct conversation
correct variant
correct workflow
correct message count
no duplicates
no cross-customer contamination
```

---

# 33. DATABASE INTEGRITY TESTING

For every major scenario verify:

```text
contact
conversation
message
campaign
variant
assignment
workflow
workflow execution
workflow step
media
stage
reply
conversion
```

relationships.

Never rely only on UI assertions.

The test must inspect the resulting persisted state.

---

# 34. MULTI-TENANT SECURITY TESTING

Deskcomm already emphasizes RLS and tenant isolation.

Do not weaken it.

Every new table must be evaluated for:

```text
tenant_id
RLS
foreign keys
authorization
auditability
```

Add isolation tests.

Tenant A must never see:

- Tenant B campaigns
- Tenant B variants
- Tenant B workflows
- Tenant B media
- Tenant B conversations
- Tenant B statistics

---

# 35. UI END-TO-END TESTING

Use Playwright or the strongest existing UI E2E tooling already present in the repository.

The QA agent must interact with the actual UI like a human.

Do not validate only through direct database manipulation.

Test:

```text
login
navigation
inbox
conversation
reply
automation creation
workflow creation
drag/drop
node configuration
media upload
save
activate
deactivate
variant creation
variant editing
variant statistics
campaign management
```

---

# 36. VISUAL WORKFLOW BUILDER QA

The QA agent must actually construct workflows through the UI.

Example:

```text
Open Automation Builder

Create workflow

Select Keyword trigger

Enter:
PROMO-A

Add Send Text

Add Wait

Add Image

Upload test image

Add Audio

Upload test audio

Add Video

Upload test video

Save

Activate
```

Then trigger the workflow through the fake Wasender provider.

Verify the complete execution.

Do not declare the builder working merely because the page renders.

---

# 37. MOBILE QA

Test at minimum:

```text
390 × 844
768 × 1024
1440 × 900
```

At mobile/tablet/desktop.

Must work on mobile:

- inbox
- conversation
- customer reply
- contact
- stage management
- tags
- activation/deactivation
- campaign statistics

Complex workflow construction may be desktop/tablet optimized.

Do not claim "responsive" merely because CSS does not overflow.

Actually click and operate the UI.

---

# 38. MEDIA QA

Test:

```text
text
image
audio
video
document
```

through:

```text
UI
 ↓
Storage
 ↓
Workflow
 ↓
Fake Wasender
```

Then through:

```text
UI
 ↓
Storage
 ↓
Workflow
 ↓
Real Wasender
 ↓
WhatsApp
```

Test invalid files too.

Test:

- wrong MIME
- oversized files
- corrupted file
- missing file
- expired URL
- duplicate upload
- deleted file
- workflow referencing unavailable media

---

# 39. REAL WHATSAPP TESTING

Only after all simulated tests pass.

Use two real Wasender sessions/numbers.

Example:

```text
NUMBER A
customer

NUMBER B
business
```

Test:

```text
A → B
B → A
```

Verify:

```text
incoming message
outgoing message
contact
conversation
session identity
message ID
workflow trigger
variant assignment
reply attribution
media
follow-up
```

Then reverse roles.

Do not claim that these tests can guarantee that WhatsApp will never ban an account.

Instead verify conservative system behavior:

- no duplicate sends
- no retry storms
- no unintended loops
- correct rate limiting
- correct delays
- correct session separation
- correct recipient
- correct sender

---

# 40. TEST CAMPAIGN

Create a controlled test campaign:

```text
Campaign:
QA-PROMO

Variant A
Variant B
Variant C
```

Each variant should have a different workflow.

Example:

```text
Variant A:
Text
→ Image
→ Wait
→ Audio

Variant B:
Text
→ Video
→ Wait
→ Image

Variant C:
Audio
→ Text
→ Wait
→ Video
```

Use test media fixtures.

Run virtual customers first.

Then run controlled real-number tests.

---

# 41. STATISTICAL TESTING

Test variant allocation.

Example:

```text
A = 50%
B = 30%
C = 20%
```

Generate thousands of virtual customers.

Verify allocation is within an acceptable statistical tolerance.

Then test:

```text
variant activation
variant deactivation
reactivation
```

Verify inactive variants receive no new assignments.

---

# 42. VERSIONED DEVELOPMENT

The implementation MUST be progressive.

Never build V1 + V2 + V3 simultaneously.

---

# VERSION 1 — MINIMUM USABLE SYSTEM

V1 must include only:

```text
Wasender adapter
basic session configuration
inbound webhook
outbound text
contacts/conversations
keyword trigger
simple workflow
send text
wait
basic media
basic variant allocation
variant activation/deactivation
basic statistics
programmatic workflow creation
fake Wasender
E2E tests
UI tests
```

V1 must be deployable and usable.

If V1 works, stop.

Do not automatically continue to V2.

The user must be able to operate V1 as a real system.

---

# VERSION 2 — ADVANCED WORKFLOW SYSTEM

Only after V1 has been used and validated.

Add:

```text
visual workflow builder
random delays
conditions
branches
advanced media
variant-specific workflows
bulk variant creation
bulk workflow import
advanced statistics
workflow execution history
better campaign analytics
```

Again:

```text
implement
→ test
→ deploy
→ validate
→ use
```

before V3.

---

# VERSION 3 — SCALE / ADVANCED EXPERIMENTATION

Only after V2 is stable.

Potentially add:

```text
210+ variants at scale
advanced statistical analysis
winner detection
traffic optimization
workflow templates
workflow duplication
advanced segmentation
advanced conversion attribution
native Wasender session management
optional Cloudflare R2
advanced mobile UX
advanced reporting
```

Do not implement V3 merely because it is possible.

---

# 43. NO BIG-BANG IMPLEMENTATION

Forbidden:

```text
"Implement everything and test at the end."
```

Required:

```text
Implement one layer
↓
Run tests
↓
Fix failures
↓
Run regression
↓
E2E
↓
UI QA
↓
Document result
↓
Only then proceed
```

---

# 44. HARD PHASE GATES

The following sequence is mandatory:

```text
PHASE 0
Repository audit
        ↓ PASS

PHASE 1
Docker/Coolify-safe baseline
        ↓ PASS

PHASE 2
Wasender provider abstraction
        ↓ PASS

PHASE 3
Fake Wasender integration
        ↓ PASS

PHASE 4
Database/message integration
        ↓ PASS

PHASE 5
Real Wasender connection
        ↓ PASS

PHASE 6
Two-number WhatsApp test
        ↓ PASS

PHASE 7
Keyword trigger
        ↓ PASS

PHASE 8
Minimal workflow
        ↓ PASS

PHASE 9
Media workflow
        ↓ PASS

PHASE 10
Visual workflow builder
        ↓ PASS

PHASE 11
Variant engine
        ↓ PASS

PHASE 12
Variant attribution/statistics
        ↓ PASS

PHASE 13
Programmatic workflow API
        ↓ PASS

PHASE 14
Bulk variants
        ↓ PASS

PHASE 15
Concurrency/idempotency
        ↓ PASS

PHASE 16
Mobile QA
        ↓ PASS

PHASE 17
Production regression
        ↓ PASS
```

If any phase fails:

**STOP.**

Do not proceed.

---

# 45. DOCKER AND COOLIFY SAFETY

The user already has a VPS and Coolify.

Do not assume a fresh server.

Before modifying anything:

```text
inspect Docker
inspect containers
inspect networks
inspect volumes
inspect ports
inspect domains
inspect reverse proxy
inspect Coolify services
inspect resource usage
inspect disk
inspect backups
inspect environment
```

Do not:

```text
docker system prune
docker volume prune
docker compose down
```

on unrelated production resources.

Never delete existing volumes without explicit confirmation.

Never replace the entire Docker environment.

Never overwrite unrelated Coolify applications.

Never bind to an already-used port without checking.

Never modify global reverse-proxy configuration without understanding existing routing.

---

# 46. DEPLOYMENT STRATEGY

Prefer:

```text
new application/service
```

or a controlled update of the existing application.

Before deployment:

```text
backup
database migration plan
rollback plan
health check
logs
resource check
```

If possible, use a staging/test deployment before touching the production application.

Every deployment must have:

```text
before state
after state
rollback procedure
```

---

# 47. DATABASE MIGRATIONS

Never manually alter production tables without a versioned migration.

Every new table must include:

- tenant ownership where applicable
- indexes
- foreign keys
- constraints
- RLS
- migration
- rollback consideration
- tests

Do not modify existing schema unnecessarily.

Before migration:

```text
backup / verify recoverability
```

After migration:

```text
schema verification
RLS verification
application tests
```

---

# 48. OBSERVABILITY

Every workflow execution should have enough information to debug:

```text
campaign
variant
workflow
workflow execution
contact
conversation
message
provider
provider session
provider message ID
stage
timestamp
status
error
```

Do not log secrets.

Do not log unnecessary personal data.

Use existing Sentry/logging infrastructure where possible.

---

# 49. FAILURE HANDLING

Test:

```text
Wasender unavailable
webhook delayed
webhook duplicated
message rejected
media upload fails
media expires
database temporarily unavailable
worker crashes
workflow is deactivated
variant is deactivated
contact deleted
conversation missing
session disconnected
```

The system must fail predictably.

Never silently lose a workflow execution.

Never silently duplicate a message.

---

# 50. REGRESSION TESTING

After every major change, verify that existing DeskcommCRM functionality still works:

```text
authentication
contacts
inbox
conversations
pipeline
tags
assignments
follow-ups
existing automations
analytics
storage
tenant isolation
```

Do not assume that because the new workflow works, the CRM is still healthy.

---

# 51. DOCUMENTATION REQUIRED

Every implemented subsystem must have documentation.

At minimum:

```text
Wasender integration
credential model
webhooks
session model
workflow model
variant model
statistics
media
programmatic API
deployment
testing
rollback
troubleshooting
```

Also create a developer document explaining:

```text
What was inherited from Deskcomm
What was adapted from WA CRM concepts
What was custom-built
What was changed in the WhatsApp engine
Why each database change exists
```

---

# 52. CODE QUALITY RULE

Do not create unnecessary abstractions.

But do create abstractions where they prevent provider lock-in.

The key boundary should be approximately:

```text
CRM
 ↓
Canonical WhatsApp interface
 ↓
Wasender implementation
```

The workflow engine should not know Wasender-specific payload formats.

The experiment engine should not know Wasender-specific payload formats.

The UI should not know Wasender API credentials.

---

# 53. FINAL ARCHITECTURE TARGET

The target architecture should resemble:

```text
                         DESKCOMM CRM
                              │
       ┌──────────────────────┼──────────────────────┐
       │                      │                      │
   Existing CRM        Workflow Engine       Experiment Engine
       │                      │                      │
 Contacts              Triggers              Campaigns
 Inbox                 Actions               Variants
 Conversations         Wait                  Assignment
 Pipeline              Conditions            Statistics
 Follow-ups            Media                 Attribution
 Tags                  Branches              Conversion
       │                      │                      │
       └──────────────────────┼──────────────────────┘
                              │
                        event_log/workers
                              │
                         Storage Layer
                              │
                     Supabase Storage V1
                              │
                       WhatsApp Adapter
                              │
                       WasenderAPI
                              │
                    WhatsApp Sessions
                              │
                    ┌─────────┴─────────┐
                    │                   │
                 Number A            Number B
```

---

# 54. PROGRAMMATIC WORKFLOW EXAMPLE

The final system should make it possible for an external coding agent to express an operation conceptually like:

```text
Create campaign "Summer Promo".

Create Variant A.

Attach:
- text message X
- audio file Y
- image Z

Workflow:
1. Send text X.
2. Wait random 60–120 seconds.
3. Send audio Y.
4. Wait 5 minutes.
5. Send image Z.

Activate Variant A.
```

The exact API/schema should be determined from the implementation.

The important requirement is:

**The external coding agent must be able to create the same workflow that a human can create visually.**

---

# 55. DEFINITION OF DONE FOR V1

V1 is complete only when all of the following are true:

- Deskcomm runs in the existing Coolify environment.
- Existing unrelated Docker applications remain untouched.
- Existing CRM functionality remains operational.
- WAHA is no longer required for the new WhatsApp path.
- Wasender authentication works correctly.
- PAT and Session API Key are never confused.
- Webhook signature verification works.
- Incoming messages reach Deskcomm.
- Outgoing text messages reach Wasender.
- Conversations are persisted.
- Duplicate webhook events are idempotent.
- Keyword trigger works.
- A basic workflow works.
- Wait works.
- Basic media works.
- A variant can be created.
- A variant can be activated/deactivated.
- A variant can be linked to a workflow.
- Variant assignment persists.
- Replies are attributed to the correct variant.
- Basic statistics are correct.
- Programmatic workflow creation works.
- Fake Wasender tests pass.
- Database integration tests pass.
- UI E2E tests pass.
- Mobile smoke tests pass.
- Real two-number test passes.
- Regression tests pass.
- Rollback procedure is documented.

---

# 56. FINAL OPERATING RULE

You are not allowed to optimize for speed at the expense of stability.

The objective is:

> **Smallest safe change → test → verify → deploy → use → learn → next version.**

Do not build an impressive but unverified system.

Build a boring, observable, testable V1.

Then improve it.

At every stage, prefer reusing existing DeskcommCRM infrastructure over recreating it.

Use WA CRM as a reference for visual workflow/automation concepts, not as a second CRM architecture. WA CRM's documented automation system and media-capable flow nodes are useful references, but Deskcomm's existing Supabase/event/worker/CRM architecture remains the foundation. ([wacrm](https://wacrm.tech/docs/automations?utm_source=chatgpt.com))

WasenderAPI must remain behind an adapter. Its current API supports session management, QR retrieval, webhooks, message sending, message information and media operations, so use those documented capabilities rather than recreating provider functionality inside Deskcomm. ([WASenderApi](https://www.wasenderapi.com/api-docs/sessions/get-whatsapp-session-qr-code?utm_source=chatgpt.com))

**Never move to the next phase when the current phase is failing.**

**Never leave the user with a partially working system that requires them to manually debug the implementation.**

When a test fails:

1. reproduce it;
2. identify the root cause;
3. fix it;
4. rerun the failing test;
5. rerun relevant regression tests;
6. rerun the E2E scenario;
7. document the fix;
8. only then continue.

The final result must be a production-oriented DeskcommCRM extension that can operate without an AI API key initially, uses WasenderAPI as its WhatsApp engine, supports visual and programmatic workflows, supports arbitrary multi-variant experiments, supports media, records complete customer context, and can be progressively evolved from V1 to V2 to V3 without requiring a big-bang rewrite.

---

# you asked

message time: 2026-08-20 12:53:53

https://wacrm.tech/llms.txt. I have not viewed the talk about the CRM automation builder because I think the automation builder UI they use is very suitable. We just need to adjust the trigger and the action to fit the API we use, so that we can match the trigger and the action we can do with the actual API we use, compared to the official API they use. Is the builder they use open source so that we can go directly to the source, or is it better to import this builder from the WA CRM project?

---

# chatgpt response

Yes. **The better approach is to take the workflow-builder implementation directly from the open-source WA CRM repository as a reference/component, not to rebuild a workflow builder from scratch.** But I would **not import the whole WA CRM automation subsystem into DeskcommCRM**.

The distinction is important.

### What I found

 is **MIT licensed**, so you are legally permitted to use, modify, and rebrand the code. The project explicitly describes itself as a self-hostable open-source CRM and says its license is MIT. ([GitHub](https://github.com/ArnasDon/wacrm?utm_source=chatgpt.com))

More importantly, its current codebase actually contains a proper visual workflow technology stack. Its dependencies include:

- `@xyflow/react`
- `@dnd-kit/core`
- `@dnd-kit/sortable`
- `@dagrejs/dagre`

So this isn't just a conceptual builder; there is actual source code for the visual workflow functionality. ([GitHub](https://github.com/ArnasDon/wacrm/blob/main/package-lock.json?utm_source=chatgpt.com))

And its newer **Flows** system has a no-code builder, node configuration, validation, activation/deactivation, persistence, and execution tracking. ([GitHub](https://github.com/ArnasDon/wacrm/blob/main/CHANGELOG.md?utm_source=chatgpt.com))

### Therefore, I would NOT do this

```text
DeskcommCRM
     ↓
copy entire WA CRM
     ↓
merge two CRMs
     ↓
try to make them work together
```

That's exactly the kind of integration that creates unnecessary complexity.

### I would do this instead

```text
                 DESKCOMMCRM
                     │
          Existing CRM architecture
                     │
              Workflow Engine
                     │
          ┌──────────┴──────────┐
          │                     │
     Visual Builder        Programmatic API
          │                     │
          └──────────┬──────────┘
                     │
             Canonical Workflow
                     │
             Execution Engine
                     │
             WhatsApp Adapter
                     │
                 Wasender
```

And take **the visual-builder layer/concepts from WA CRM**.

---

# The important discovery: WA CRM already separates the builder from WhatsApp

This is actually very favorable for your project.

WA CRM's automation system is built around concepts such as:

- inbound-message triggers
- keyword triggers
- new-contact triggers
- scheduled triggers
- waits
- conditions
- tags
- webhooks
- branching
- workflow execution

The project's documentation explicitly describes its automations as event-driven step chains, including keyword auto-replies and other triggers. ([wacrm](https://wacrm.tech/llms.txt))

That means your intuition is basically correct:

> **We don't need to reinvent the workflow-builder UX. We need to make the workflow engine understand the events and actions that WasenderAPI actually provides.**

---

# But there is one major caveat

WA CRM is designed around the **official Meta WhatsApp Cloud API**, whereas you are replacing Deskcomm's WhatsApp engine with **WasenderAPI**.

WA CRM itself explicitly identifies Meta Cloud API as its WhatsApp channel. ([GitHub](https://github.com/ArnasDon/wacrm?utm_source=chatgpt.com))

Therefore, don't copy its triggers/actions blindly.

For example:

```text
WA CRM trigger
      ↓
Meta Cloud API webhook
```

needs to become:

```text
Wasender webhook
      ↓
Wasender adapter
      ↓
Canonical Deskcomm event
      ↓
WA-CRM-style workflow trigger
```

This is exactly why the **adapter layer** I proposed in the master prompt is important.

---

# The builder should NOT know WasenderAPI

This is probably the most important architectural decision.

The visual builder should say:

> **When customer sends keyword `PROMO`**

not:

> When Wasender event `messages.upsert` with payload X occurs.

Likewise, the action should say:

> **Send audio**

not:

> Call Wasender `/api/send-message` with payload X.

So:

```text
              VISUAL BUILDER
                    ↓
             Canonical Actions
                    ↓
             Workflow Engine
                    ↓
            WhatsApp Interface
                    ↓
             Wasender Adapter
                    ↓
               WasenderAPI
```

That gives you tremendous flexibility.

If you later replace Wasender with another provider, you don't rebuild your workflow builder.

---

# Should we literally copy WA CRM's builder code?

**Potentially yes — but only after the coding agent performs a component-level audit.**

I would instruct the agent to identify exactly:

```text
/components
/workflows
/automations
/flows
/hooks
/API routes
/database models
```

and determine which pieces are:

### A. Pure UI

Good candidates for reuse/adaptation.

For example:

```text
canvas
nodes
node editor
drag/drop
connections
toolbar
sidebar
validation UI
```

### B. Workflow-domain logic

Potentially reusable, but must be adapted.

For example:

```text
workflow schema
node schema
execution state
conditions
wait logic
branching
```

### C. Meta-specific WhatsApp logic

**Do not import this into Deskcomm.**

For example:

```text
Meta webhook parsing
Meta template APIs
Meta-specific delivery handling
Meta Cloud API authentication
Meta-specific message payloads
```

Those should be replaced with your Wasender adapter.

---

# There's another very important advantage

WA CRM already uses a modern visual graph stack:

```text
@xyflow/react
```

combined with drag/drop tooling and Dagre layout. ([GitHub](https://github.com/ArnasDon/wacrm/blob/main/package-lock.json?utm_source=chatgpt.com))

So your coding agent shouldn't waste time designing:

> "How do we draw boxes and connect them?"

That problem has already been solved.

Instead, your engineering effort should go toward:

```text
Wasender events
        ↓
Canonical events
        ↓
Workflow triggers
        ↓
Workflow execution
        ↓
Variant assignment
        ↓
Statistics
        ↓
Attribution
```

That's where your actual product differentiation is.

---

# And your programmatic requirement fits beautifully

This is where I would go beyond simply copying the WA CRM UI.

The **visual builder and programmatic API should generate exactly the same workflow representation.**

For example:

```text
                  Workflow Definition
                         │
             ┌───────────┴───────────┐
             ↓                       ↓
       Visual Builder          Programmatic API
             │                       │
             └───────────┬───────────┘
                         ↓
                  Workflow Engine
```

So your AI coding agent could eventually do something like:

```text
Create workflow "Variant A"

Trigger:
keyword = "PROMO"

Steps:
1. Send text
2. Wait 60-120 seconds
3. Send audio /audio/promo.mp3
4. Wait 5 minutes
5. Send image /images/product.jpg
```

And the visual UI should show **exactly that workflow** afterward.

Conversely, if you create it visually, the API should return the same structure.

That is much more powerful than merely copying WA CRM.

---

# My recommendation for your project

I would change one part of the master prompt.

Instead of saying:

> "Build a visual workflow builder inspired by WA CRM."

I would make it:

> **"Perform a component-level audit of the MIT-licensed WA CRM workflow/automation builder. Reuse/adapt its visual builder components where technically compatible with DeskcommCRM. Do not import the WA CRM CRM architecture, database, authentication, Meta integration, or WhatsApp transport layer. Extract only the minimum reusable UI/graph-builder components and adapt them to Deskcomm's existing workflow representation. All triggers/actions must operate through Deskcomm's canonical event/action interfaces, with WasenderAPI isolated behind the WhatsApp provider adapter."**

That's much more precise.

### Final architecture

```text
                 WA CRM SOURCE
                      │
             ┌────────┴─────────┐
             │                  │
        Visual Builder       UX patterns
             │                  │
             └────────┬─────────┘
                      ↓
                DESKCOMMCRM
                      │
          ┌───────────┼────────────┐
          ↓           ↓            ↓
      CRM         Workflow     Experiment
                  Engine        Engine
                      │
                Canonical Events
                      │
               Wasender Adapter
                      │
                  WasenderAPI
                      │
              WhatsApp Sessions
```

**So: don't import WA CRM wholesale. Don't rebuild the builder. Reuse the open-source builder technology/components and adapt its trigger/action/execution boundaries to Deskcomm + Wasender.**

And because WA CRM is MIT licensed, this is technically and legally much more attractive than trying to reproduce the UI from scratch. ([wacrm](https://wacrm.tech/llms.txt))

One additional point: WA CRM's newer **Flows** system is more sophisticated than the older linear Automations system—it has branching, a state machine, active-run handling, and a dedicated builder. ([GitHub](https://github.com/ArnasDon/wacrm/blob/main/CHANGELOG.md?utm_source=chatgpt.com)) **That newer implementation is the part I would have your coding agent inspect first**, rather than copying an older/simple automation UI.

---

# you asked

message time: 2026-08-20 12:55:15

Adjust the master prompt to incorporate this in a way that is meticulous.

---

# chatgpt response

Absolutely. The master prompt should now explicitly instruct the coding agent to **reuse the WA CRM visual workflow-builder technology where appropriate**, while **not importing its CRM, database, Meta/WhatsApp transport, or execution architecture wholesale**.

I would also make the agent perform a **source-code/component audit before deciding what to copy**, because "import the builder" can mean anything from reusing a few React components to accidentally importing half of another CRM.

Here is the revised master prompt.

---

# MASTER ENGINEERING PROMPT
## DeskcommCRM → WasenderAPI + WA-CRM Workflow Builder + Keyword Automation + Multi-Variant Experimentation

### Mission

You are an autonomous senior software architect, full-stack engineer, DevOps engineer, database engineer, QA engineer, security engineer, and Playwright/UI automation engineer.

Your task is to extend:

**DeskcommCRM**

Repository:

`https://github.com/melgarafael/DeskcommCRM`

with:

1. **WasenderAPI as the WhatsApp transport layer**
2. **A robust keyword-triggered workflow automation engine**
3. **A visual workflow builder based on the open-source WA CRM builder where technically appropriate**
4. **Programmatic workflow creation**
5. **Multi-variant experimentation with arbitrary numbers of variants**
6. **Variant-level statistics and reply attribution**
7. **Media-aware workflows**
8. **Progressive V1 → V2 → V3 development**
9. **Full automated testing before real WhatsApp testing**
10. **Safe deployment to the user's existing Coolify/VPS environment**

The fundamental philosophy is:

> **Do not rebuild what DeskcommCRM already does well. Do not rebuild the workflow builder from scratch if an appropriate open-source implementation already exists. Do not import an entire second CRM. Build the smallest possible set of additions and adapters necessary to turn DeskcommCRM into the desired system.**

---

# 1. THE THREE PRIMARY ARCHITECTURAL CHANGES

The project has three fundamental changes.

### Change 1 — WhatsApp engine

Replace/adapt:

```text
WAHA
```

with:

```text
WasenderAPI
```

### Change 2 — Automation/workflow engine

Add a robust:

```text
Keyword
→ Workflow
→ Delays
→ Conditions
→ Messages
→ Media
→ Branches
```

system.

Where possible, reuse/adapt the **open-source WA CRM workflow-builder UI and graph technology** instead of implementing an entirely new visual builder.

### Change 3 — AI remains optional

Do NOT remove DeskcommCRM's AI architecture.

Do NOT require an AI API key for V1.

Do NOT make workflow execution dependent on AI.

The user simply does not intend to configure an AI API key initially.

The system must remain architecturally capable of adding AI later.

---

# 2. CRITICAL ARCHITECTURAL PRINCIPLE

There are now **four separate layers** that must not be confused.

```text
┌─────────────────────────────────────────┐
│             DESKCOMM CRM                │
│ Contacts / Inbox / Pipeline / CRM       │
└────────────────────┬────────────────────┘
                     │
┌────────────────────▼────────────────────┐
│       WORKFLOW / AUTOMATION LAYER       │
│ Triggers / Nodes / Delays / Branches   │
└────────────────────┬────────────────────┘
                     │
┌────────────────────▼────────────────────┐
│      CANONICAL WHATSAPP INTERFACE       │
│ Provider-independent events/actions     │
└────────────────────┬────────────────────┘
                     │
┌────────────────────▼────────────────────┐
│          WASENDER ADAPTER               │
│ Wasender-specific API implementation    │
└────────────────────┬────────────────────┘
                     │
┌────────────────────▼────────────────────┐
│             WASENDER API                │
└────────────────────┬────────────────────┘
                     │
                  WhatsApp
```

The **visual builder must not directly depend on WasenderAPI**.

The **workflow engine must not directly parse Wasender webhook payloads**.

The **CRM must not know Wasender-specific API details**.

This separation is mandatory.

---

# 3. SOURCE PROJECTS TO ANALYZE

Before coding, inspect all three sources deeply.

### Primary application

`https://github.com/melgarafael/DeskcommCRM`

### Workflow-builder reference

`https://github.com/ArnasDon/wacrm`

### WasenderAPI

`https://www.wasenderapi.com/llms.txt`

Also inspect the relevant WasenderAPI API documentation.

Do not rely solely on README files.

Inspect:

- source code
- package dependencies
- database schema
- API routes
- components
- hooks
- workflow implementation
- webhook handlers
- tests
- Docker configuration
- deployment configuration
- environment variables
- storage implementation
- worker implementation
- authentication
- RLS
- existing automation functionality

---

# 4. MANDATORY WA CRM BUILDER AUDIT

Before deciding what to reuse from WA CRM, perform a **component-level audit**.

Do not simply say:

> "WA CRM has a workflow builder."

Determine exactly what exists.

Identify:

```text
workflow builder UI
canvas
nodes
edges
drag/drop
node editor
sidebar
toolbar
workflow persistence
workflow schema
validation
execution engine
state machine
trigger system
action system
conditions
branching
wait implementation
media handling
automation API
database models
Meta integration
WhatsApp integration
```

Classify every component into four categories.

---

## CATEGORY A — PURE VISUAL COMPONENTS

Examples:

```text
workflow canvas
node rendering
edge rendering
drag/drop
node selection
node configuration panel
workflow toolbar
zoom/pan
layout
visual validation
```

These are strong candidates for direct reuse/adaptation.

---

## CATEGORY B — PROVIDER-INDEPENDENT WORKFLOW LOGIC

Examples:

```text
workflow schema
node schema
branching
conditions
wait state
workflow validation
execution state
```

These may be reusable, but compare them against Deskcomm's existing architecture before importing them.

---

## CATEGORY C — WA CRM-SPECIFIC BUSINESS LOGIC

Examples:

```text
WA CRM database models
WA CRM authentication
WA CRM contacts
WA CRM campaigns
WA CRM tenant architecture
WA CRM-specific execution architecture
```

Do NOT import these automatically.

Map their functionality against Deskcomm's existing systems first.

If Deskcomm already has the capability, reuse Deskcomm.

---

## CATEGORY D — META/WHATSAPP-SPECIFIC CODE

Examples:

```text
Meta Cloud API
Meta webhook payloads
Meta access tokens
Meta templates
Meta-specific media APIs
Meta-specific message structures
```

Do NOT import this.

Your WhatsApp transport is:

```text
WasenderAPI
```

---

# 5. IMPORTANT: DO NOT "MERGE TWO CRMs"

Forbidden architecture:

```text
DeskcommCRM
+
WA CRM
+
Wasender
=
three systems glued together
```

The target architecture is:

```text
DeskcommCRM
      +
selected workflow-builder components
      +
custom automation/experiment layer
      +
Wasender adapter
```

WA CRM is a **source of reusable workflow-builder technology**, not a second CRM that should run alongside Deskcomm.

---

# 6. WHY THE WA CRM BUILDER IS BEING USED

The WA CRM project is MIT licensed.

Its source should therefore be inspected directly and reused where technically appropriate.

Its workflow technology includes modern graph/drag-and-drop tooling such as:

```text
@xyflow/react
@dnd-kit/core
@dnd-kit/sortable
@dagrejs/dagre
```

Do not recreate this functionality unnecessarily.

The goal is:

> **Reuse proven workflow-builder mechanics; replace/adapt the business logic underneath them.**

---

# 7. WORKFLOW BUILDER TARGET

The final UI should allow a human to construct:

```text
Keyword Trigger
      ↓
Send Text
      ↓
Wait
      ↓
Send Image
      ↓
Wait Random 60–180 sec
      ↓
Send Audio
      ↓
Condition
    ↙   ↘
 YES     NO
 ↓       ↓
Video   Follow-up
```

The same workflow must also be constructible programmatically.

---

# 8. VISUAL BUILDER AND PROGRAMMATIC API MUST SHARE ONE MODEL

This is mandatory.

Do NOT create:

```text
Visual Workflow Model
```

and separately:

```text
Programmatic Workflow Model
```

Instead:

```text
                 Workflow Definition
                  /              \
                 /                \
         Visual Builder          API
                 \                /
                  \              /
                   Execution
```

The UI and API must create/update the exact same underlying workflow representation.

Therefore:

### Human

creates workflow visually.

### AI coding agent

creates workflow through API.

### Result

Both produce identical executable workflows.

---

# 9. PROGRAMMATIC WORKFLOW CREATION

The final system must allow an external coding agent to issue operations conceptually like:

```text
Create campaign "PROMO-2026"

Create Variant A

Create workflow:

Trigger:
keyword = "PROMO"

Step 1:
send text

Step 2:
wait random 60–120 seconds

Step 3:
send image

Step 4:
send audio

Step 5:
wait 5 minutes

Step 6:
send video

Attach workflow to Variant A

Activate Variant A
```

The exact API structure must follow the application's architecture.

Do not invent an unrelated API framework.

---

# 10. WASENDER ADAPTER

Create a dedicated provider adapter.

Conceptually:

```text
WhatsAppProvider
      │
      └── WasenderProvider
```

The rest of the application interacts with:

```text
sendText()
sendImage()
sendAudio()
sendVideo()
sendDocument()
getSessionStatus()
```

or equivalent canonical operations.

The rest of Deskcomm must not need to know the Wasender HTTP implementation.

---

# 11. WASENDER PAT VS SESSION API KEY

This distinction is mandatory.

Never confuse:

### Personal Access Token

Account-level credential.

Used for account/session-management operations supported by Wasender.

### Session API Key

Session-scoped credential.

Used for operations associated with a particular WhatsApp session.

Conceptually:

```text
Wasender Account
      │
      └── Personal Access Token
             │
             ├── Session A
             │      └── Session API Key A
             │
             ├── Session B
             │      └── Session API Key B
             │
             └── Session C
                    └── Session API Key C
```

The implementation must explicitly document which credential each endpoint requires.

Never guess.

Never expose PATs or session keys to the browser.

Never place secrets in `NEXT_PUBLIC_*`.

Never log them.

---

# 12. QR-CODE / SESSION CONNECTION

Do not automatically reproduce WAHA's QR architecture.

Investigate Wasender's supported session-management model first.

Determine whether V1 should:

### Option A

Connect sessions through the Wasender dashboard.

or:

### Option B

Allow Deskcomm to retrieve/display the QR through the Wasender API.

Use the lowest-complexity reliable solution for V1.

Native QR management can be V2/V3 if necessary.

---

# 13. CANONICAL WHATSAPP EVENT MODEL

Normalize Wasender events:

```text
Wasender Webhook
       ↓
Parser
       ↓
CanonicalWhatsAppEvent
```

Canonical event should include only information Deskcomm requires:

```text
provider
session
message ID
contact
conversation
direction
type
text
media
timestamp
sender
recipient
quoted message
raw event reference
```

The workflow engine consumes the canonical event.

Not Wasender's raw payload.

---

# 14. WEBHOOK PROCESSING

Required:

```text
Webhook
 ↓
verify signature
 ↓
idempotency check
 ↓
acknowledge
 ↓
persist event
 ↓
queue processing
 ↓
workflow execution
```

Do not execute long workflows synchronously inside the webhook request.

Wasender-specific webhook handling must be isolated inside the provider layer.

---

# 15. IDEMPOTENCY

Duplicate webhook:

```text
same message ID
same event
```

must produce:

```text
one message
one trigger
one workflow execution
zero duplicate outgoing messages
```

Test concurrency as well as sequential duplication.

---

# 16. KEYWORD AUTOMATION

V1 trigger:

```text
incoming WhatsApp message
        +
keyword match
```

Examples:

```text
PROMO
PRICE
INFO
START
```

Define keyword matching rules explicitly.

Support exact/normalized matching according to product requirements.

Do not hardcode French or English.

The system must support arbitrary languages.

---

# 17. WORKFLOW ACTIONS

V1:

```text
Send text
Send image
Send audio
Send video
Send document
Wait
Stop
```

V2:

```text
Random wait
Condition
Branch
Tag
Pipeline stage
Assignment
Webhook
Contact update
```

V3:

```text
advanced segmentation
reusable subflows
advanced experimentation
conversion optimization
```

---

# 18. RANDOM DELAY

Support:

```text
min = 60 sec
max = 180 sec
```

Execution randomly chooses a value within that range.

Tests must support deterministic random seeds.

Never make tests wait three real minutes.

Use fake clocks/time injection.

---

# 19. WORKFLOW DURABILITY

A waiting workflow must survive:

- process restart
- Docker restart
- deployment
- worker restart
- temporary API failure

It must resume from its persisted state.

It must not restart from step one.

It must not duplicate previous messages.

Use Deskcomm's existing worker/event architecture wherever possible.

---

# 20. MULTI-VARIANT EXPERIMENTATION

Support:

```text
Variant 1
Variant 2
Variant 3
...
Variant N
```

210 variants must work without redesigning the database.

Each variant:

```text
id
campaign
name
status
weight
workflow
created_at
updated_at
```

---

# 21. VARIANT STATUS

Each variant:

```text
ACTIVE
INACTIVE
```

must be independently controllable.

Deactivating a variant must prevent new assignments.

Existing assignments must remain historically attributable.

Never delete historical experiment data merely because a variant is deactivated.

---

# 22. VARIANT ASSIGNMENT

When a customer triggers an experiment:

```text
Customer
 ↓
Campaign
 ↓
Allocator
 ↓
Variant
```

Persist:

```text
customer
campaign
variant
workflow
assignment timestamp
```

A customer must remain associated with the same variant for that experiment unless explicitly reassigned.

---

# 23. VARIANT-SPECIFIC WORKFLOWS

Example:

```text
Variant A → Workflow A

Variant B → Workflow B

Variant C → Workflow C
```

Workflows may be completely different.

---

# 24. VARIANT STATISTICS

At minimum:

```text
assigned
sent
delivered if provider confirms
replies
reply rate
workflow started
workflow completed
workflow stopped
conversion
conversion rate
average response time
```

Never fabricate provider-level metrics.

If Wasender doesn't expose a metric, mark it appropriately as unavailable.

---

# 25. REPLY ATTRIBUTION

Example:

```text
Customer X
 ↓
Campaign Y
 ↓
Variant B
 ↓
Workflow B
 ↓
Message B17
 ↓
Customer reply
```

The reply must be attributable to:

```text
Campaign Y
Variant B
Workflow B
```

This is fundamental to the experiment engine.

---

# 26. MEDIA

Use Deskcomm's existing storage architecture first.

Do not introduce Cloudflare R2 merely because it is available.

V1:

```text
Supabase Storage
```

If future scale requires R2:

```text
StorageProvider
   ├── Supabase
   └── Cloudflare R2
```

Do not redesign the workflow engine to change storage providers.

---

# 27. MEDIA WORKFLOW

The builder must support:

```text
Upload media
 ↓
Storage
 ↓
Attach to workflow node
 ↓
Execute workflow
 ↓
Resolve media
 ↓
Wasender
```

Test:

- image
- audio
- video
- document

and failures.

---

# 28. TESTING ARCHITECTURE

Testing must happen in layers.

```text
Unit
 ↓
Integration
 ↓
Contract
 ↓
Fake Wasender
 ↓
Workflow state machine
 ↓
Database
 ↓
Concurrency
 ↓
UI E2E
 ↓
Real Wasender
 ↓
Real WhatsApp
```

Do not jump directly to WhatsApp.

---

# 29. FAKE WASENDER PROVIDER

Create a deterministic fake provider implementing the same canonical interface.

It must simulate:

```text
incoming message
outgoing message
image
audio
video
document
delivery
failure
duplicate webhook
session disconnect
```

This becomes the primary automated testing environment.

---

# 30. INTELLIGENT WORKFLOW TESTING

Do not merely test:

> Button works.

Construct realistic workflows.

Example:

```text
Variant A

Keyword
 ↓
Text
 ↓
Wait
 ↓
Image
 ↓
Wait
 ↓
Audio
```

Variant B:

```text
Keyword
 ↓
Audio
 ↓
Wait
 ↓
Video
```

Variant C:

```text
Keyword
 ↓
Text
 ↓
Condition
 ├── YES → Image → Audio
 └── NO  → Video
```

Run them against simulated customers.

Verify database state after execution.

---

# 31. STATE-MACHINE TESTING

Test every state:

```text
TRIGGERED
RUNNING
WAITING
RESUMING
COMPLETED
FAILED
CANCELLED
```

Test transitions.

Test restart recovery.

Test duplicate events.

Test provider failure.

---

# 32. PROPERTY-BASED TESTING

Generate many customers and events.

Verify invariants:

```text
no duplicate workflow execution

no duplicate outgoing message

no cross-customer contamination

no cross-variant attribution

inactive variant receives no new assignments

workflow execution cannot jump to another workflow

session A cannot send using session B

tenant A cannot access tenant B
```

---

# 33. DATABASE TESTING

Verify persisted relationships:

```text
contact
conversation
message
campaign
variant
assignment
workflow
workflow execution
workflow step
media
reply
conversion
```

Do not rely exclusively on UI assertions.

---

# 34. VISUAL BUILDER E2E TESTING

The QA agent must use the actual UI.

Example:

```text
Login

Open Automations

Create Workflow

Drag Keyword Trigger

Configure keyword

Drag Send Text

Configure text

Drag Wait

Configure delay

Drag Image

Upload image

Drag Audio

Upload audio

Connect nodes

Save

Activate
```

Then trigger it through the fake provider.

Verify actual execution.

---

# 35. PROGRAMMATIC BUILDER TESTING

Create the exact same workflow through the API.

Compare:

```text
Visual Workflow
vs
Programmatic Workflow
```

They must result in equivalent workflow definitions.

This test is mandatory.

---

# 36. BULK CREATION TEST

Generate:

```text
210 variants
```

programmatically.

Each variant must have:

```text
workflow
message
media
assignment
```

Verify:

- database integrity
- performance
- UI rendering
- statistics
- activation/deactivation
- assignment
- attribution

The UI must not attempt to render 210 massive workflows simultaneously.

Use pagination/virtualization where appropriate.

---

# 37. UI QA

Use Playwright.

Test:

### Desktop

```text
1440×900
```

### Tablet

```text
768×1024
```

### Mobile

```text
390×844
```

Test actual interaction.

---

# 38. MOBILE REQUIREMENTS

Mobile must support:

- inbox
- conversation
- reply
- contacts
- pipeline
- stage changes
- variant status
- campaign statistics

Complex workflow creation may be optimized primarily for desktop/tablet.

Do not waste V1 engineering effort making a complex graph editor perfect on a 6-inch screen unless testing demonstrates it is required.

---

# 39. DOCKER / COOLIFY SAFETY

The user already has:

```text
existing VPS
existing Docker environment
existing Coolify
existing applications
```

This is NOT a fresh-server deployment.

Before touching anything:

```text
Docker inventory
containers
networks
volumes
ports
disk
RAM
CPU
Coolify services
reverse proxy
domains
environment variables
```

Never execute destructive cleanup commands against the environment.

Do not blindly run:

```text
docker system prune
docker volume prune
docker compose down
```

Do not touch unrelated applications.

---

# 40. DEPLOYMENT

Prefer a controlled Coolify deployment.

Before deployment:

```text
backup
migration plan
rollback plan
health check
resource check
```

After deployment:

```text
health check
logs
database verification
API verification
UI E2E
```

---

# 41. VERSIONED IMPLEMENTATION

The system MUST be developed progressively.

---

## V1 — MINIMUM USABLE

Implement only:

```text
Wasender adapter
webhook
text sending
basic media
keyword trigger
simple workflow
wait
basic variant engine
variant activation/deactivation
basic statistics
programmatic workflow creation
fake provider
automated tests
basic UI builder
```

V1 must actually be usable.

STOP after V1.

Do not automatically implement V2.

---

## V2 — ADVANCED WORKFLOW

Only after V1 has been validated in real use.

Add:

```text
full visual graph builder
random delays
conditions
branches
advanced media
variant-specific workflows
bulk workflow creation
advanced analytics
execution history
```

STOP.

Validate again.

---

## V3 — SCALE

Only after V2 is stable.

Potentially add:

```text
210+ variant optimization
advanced statistical analysis
winner detection
traffic optimization
workflow templates
subflows
advanced segmentation
R2
native QR/session management
advanced mobile UX
```

Only implement features that have demonstrated value.

---

# 42. PHASE GATES

Never proceed automatically.

```text
PHASE 0
Repository audit
        ↓
PASS

PHASE 1
WAHA → Wasender adapter
        ↓
PASS

PHASE 2
Fake provider
        ↓
PASS

PHASE 3
Canonical events
        ↓
PASS

PHASE 4
Keyword automation
        ↓
PASS

PHASE 5
Basic workflow
        ↓
PASS

PHASE 6
Media
        ↓
PASS

PHASE 7
Visual builder
        ↓
PASS

PHASE 8
Variants
        ↓
PASS

PHASE 9
Statistics + attribution
        ↓
PASS

PHASE 10
Programmatic workflow API
        ↓
PASS

PHASE 11
Real Wasender
        ↓
PASS

PHASE 12
Two real WhatsApp numbers
        ↓
PASS

PHASE 13
Production deployment
        ↓
PASS
```

If a phase fails:

**STOP.**

Fix it before proceeding.

---

# 43. REQUIRED WA CRM SOURCE-CODE DELIVERABLE

Before implementation, produce a table:

| WA CRM Component | Location | Type | Reuse? | Reason |
|---|---|---|---|---|
| Workflow canvas | ... | UI | YES/NO | ... |
| Node system | ... | UI/domain | YES/NO | ... |
| Drag/drop | ... | UI | YES/NO | ... |
| Edge system | ... | UI | YES/NO | ... |
| Workflow schema | ... | domain | YES/NO | ... |
| Executor | ... | backend | YES/NO | ... |
| Wait system | ... | backend | YES/NO | ... |
| Conditions | ... | domain | YES/NO | ... |
| Meta integration | ... | provider | NO | Replace with Wasender |
| Database | ... | infrastructure | NO | Deskcomm already has DB |

Do not begin copying components until this audit exists.

---

# 44. REQUIRED DESKCOMM SOURCE-CODE DELIVERABLE

Produce another matrix:

| Deskcomm capability | Existing? | Reuse? | Modify? | New? |
|---|---:|---:|---:|---:|
| Contacts | | | | |
| Conversations | | | | |
| Messages | | | | |
| Pipelines | | | | |
| Follow-ups | | | | |
| Workers | | | | |
| event_log | | | | |
| Storage | | | | |
| Automation | | | | |
| Workflow | | | | |
| AI | | | | |
| WAHA | | | | |

The purpose is to prevent reinventing functionality that already exists.

---

# 45. REQUIRED WASENDER COMPATIBILITY MATRIX

Create:

| Capability | WAHA | Wasender | Deskcomm impact |
|---|---|---|---|
| Session creation | | | |
| QR | | | |
| Session status | | | |
| Incoming text | | | |
| Outgoing text | | | |
| Image | | | |
| Audio | | | |
| Video | | | |
| Document | | | |
| Webhook | | | |
| Delivery status | | | |
| Message ID | | | |
| Media | | | |
| Errors | | | |

No assumption is allowed.

Verify against documentation and source/API behavior.

---

# 46. NO AI API KEY

The initial system must work without configuring an AI provider wherever the existing Deskcomm application permits.

AI remains available for future configuration.

Do not remove:

```text
AI SDK
AI interfaces
AI configuration
```

unless the repository audit proves they are unnecessary and their removal is explicitly justified.

The user can later provide an AI provider.

---

# 47. SECURITY

Mandatory:

- server-side secrets
- webhook verification
- RLS
- tenant isolation
- authorization
- input validation
- file validation
- rate limiting
- idempotency
- audit logs
- no token leakage

---

# 48. FINAL ARCHITECTURE

The desired result is:

```text
                    DESKCOMM CRM
                         │
        ┌────────────────┼────────────────┐
        │                │                │
       CRM          Workflow Engine   Experiments
        │                │                │
        │                │                │
        │         ┌──────┴──────┐         │
        │         │             │         │
        │      Visual UI      API         │
        │         │             │         │
        │         └──────┬──────┘         │
        │                │                │
        └────────────────┼────────────────┘
                         │
                  Canonical Events
                         │
                 WhatsApp Interface
                         │
                  Wasender Adapter
                         │
                    WasenderAPI
                         │
              ┌──────────┴──────────┐
              │                     │
          Session A             Session B
              │                     │
              └──────── WhatsApp ───┘
```

The visual workflow layer may reuse/adapt WA CRM's open-source workflow-builder technology.

The business logic remains Deskcomm.

The WhatsApp provider becomes Wasender.

The experimentation layer is custom.

The workflow API and visual builder share the same workflow representation.

---

# 49. DEFINITION OF DONE

The implementation is complete only when:

### Architecture

- Deskcomm remains the primary CRM.
- WA CRM was not merged as a second CRM.
- WAHA has been replaced/adapted through a provider boundary.
- Wasender-specific logic is isolated.
- Workflow UI and API share one workflow model.

### Workflow

- keyword trigger works
- text works
- media works
- wait works
- random wait works when enabled
- conditions work
- branching works
- workflow persistence works
- workflow recovery works

### Experimentation

- arbitrary variants
- activate/deactivate
- variant-specific workflows
- assignment persistence
- statistics
- reply attribution
- conversion attribution

### Programmatic automation

- workflows can be created through API
- workflows can be modified through API
- media can be attached programmatically
- variants can be created programmatically
- bulk creation works

### Testing

- unit tests
- integration tests
- fake Wasender
- state-machine tests
- property-based tests
- concurrency tests
- database tests
- Playwright UI tests
- visual-builder E2E
- mobile QA
- real Wasender
- two-number WhatsApp test

### Deployment

- existing Coolify environment preserved
- unrelated Docker applications untouched
- rollback documented
- migrations versioned
- health checks implemented

---

# 50. FINAL RULE

The guiding principle for every implementation decision is:

> **Reuse first. Adapt second. Extend third. Rewrite only when absolutely necessary.**

Before creating anything new, search DeskcommCRM.

Before creating a visual workflow feature, inspect WA CRM.

Before implementing a WhatsApp feature, inspect WasenderAPI.

Before modifying the database, inspect the existing schema.

Before changing Docker, inspect the existing Coolify environment.

Before proceeding to the next feature, test the current feature.

Before using real WhatsApp, prove the system with a simulated provider.

Before production deployment, prove the complete system with two controlled WhatsApp numbers.

Never build the entire system and test it at the end.

The development cycle is permanently:

```text
ANALYZE
 ↓
MINIMAL IMPLEMENTATION
 ↓
UNIT TEST
 ↓
INTEGRATION TEST
 ↓
E2E TEST
 ↓
UI QA
 ↓
REGRESSION
 ↓
DOCUMENT
 ↓
DEPLOY
 ↓
OBSERVE
 ↓
ONLY THEN ADD COMPLEXITY
```

**Do not optimize for the number of features implemented. Optimize for the number of features that are actually proven to work.**

---

# you asked

message time: 2026-08-20 12:56:28

You forgot something. Just don't modify, but add it so that I can add it at the bottom of the prompt. Expect what is missing from your previous prompt and adjust it.

---

# chatgpt response

Yes. The previous prompt is missing several important controls that we discussed earlier. **Do not modify the previous prompt; append the following section verbatim at the bottom.**

---

# 51. ADDITIONAL MANDATORY REQUIREMENTS — DO NOT OMIT

The following requirements are **additive** to everything above. They must be treated as hard requirements.

## 51.1 EXISTING USER DATA AND DATABASE SAFETY

The existing DeskcommCRM database is valuable and must be treated as production data.

Before any schema modification:

1. Inspect the complete existing schema.
2. Identify all existing tables, relationships, indexes, constraints, triggers, RLS policies, and migrations.
3. Identify which tables are actively used by the current application.
4. Do not rename, delete, or repurpose existing columns/tables unless absolutely necessary.
5. Prefer additive migrations.
6. New functionality should use new tables/modules where possible.
7. Existing records must remain valid after every migration.
8. Every migration must have a rollback strategy.
9. Test migrations against a copy/snapshot before production.
10. Never run destructive database commands automatically.

The principle is:

> **Extend the existing database; do not destabilize it.**

---

# 51.2 NO UNNECESSARY REFACTORING

The coding agent must NOT use this project as an opportunity to "clean up" the existing codebase.

Do not:

- rewrite working components
- rename unrelated files
- replace frameworks
- upgrade major dependencies unnecessarily
- restructure the entire project
- rewrite authentication
- rewrite the CRM
- rewrite the existing inbox
- rewrite working database logic
- change unrelated Docker services

Every modification must answer:

> **What exact requirement does this change satisfy?**

If the answer is unclear, do not make the change.

---

# 51.3 CHANGE IMPACT ANALYSIS BEFORE EVERY MODIFICATION

Before modifying an existing component, generate a short internal impact analysis:

```text
Component:
Current responsibility:
Why modification is required:
Files affected:
Database affected:
API affected:
Potential regression:
How it will be tested:
Rollback:
```

The agent must understand the dependency graph before modifying code.

---

# 51.4 GIT CHECKPOINTS

Create a Git checkpoint after every successfully validated phase.

Example:

```text
v1.0-baseline
v1.1-wasender-adapter
v1.2-webhooks
v1.3-keyword-engine
v1.4-workflow-engine
v1.5-media
v1.6-variants
v1.7-statistics
```

Do not create a checkpoint merely because code compiles.

A checkpoint should represent:

> **implemented + tested + verified**

If a later feature breaks the system, immediately identify the last known-good checkpoint.

---

# 51.5 GOLDEN BASELINE

Before modifying DeskcommCRM:

1. Deploy/run the original application.
2. Verify its existing functionality.
3. Run the existing tests.
4. Perform a baseline UI walkthrough.
5. Record the current health state.
6. Record current Docker/Coolify configuration.
7. Record current database migration state.

This becomes the **golden baseline**.

After every major phase, regression-test against this baseline.

The question is always:

> **Did our new feature work without breaking something that already worked?**

---

# 51.6 FULL INVENTORY BEFORE CODING

Before writing implementation code, inspect:

```text
Repository
├── frontend
├── backend
├── API
├── database
├── migrations
├── workers
├── queues
├── webhooks
├── authentication
├── storage
├── Docker
├── Coolify configuration
├── environment configuration
├── existing automation
├── existing AI integration
├── existing WhatsApp integration
└── tests
```

Produce an architecture map.

Do not begin implementation based only on README documentation.

---

# 51.7 DO NOT ASSUME THE EXISTING WAHA IMPLEMENTATION

The agent must determine exactly how WAHA is currently integrated.

Trace:

```text
WAHA
 ↓
configuration
 ↓
API client
 ↓
webhooks
 ↓
message normalization
 ↓
database
 ↓
conversation
 ↓
automation
 ↓
outgoing message
```

Then determine which portions can remain unchanged.

The objective is NOT:

> "Replace every WAHA-related file."

The objective is:

> **Replace only the provider-specific transport functionality while preserving the application's higher-level behavior.**

---

# 51.8 WASENDER CONTRACT-FIRST IMPLEMENTATION

Before implementing Wasender integration, create an explicit provider contract.

For example:

```text
WhatsAppProvider
├── sendText()
├── sendImage()
├── sendAudio()
├── sendVideo()
├── sendDocument()
├── getSessionStatus()
├── handleWebhook()
└── normalizeEvent()
```

The exact interface must be derived from Deskcomm's existing architecture and Wasender's actual API.

Do not invent methods that Wasender cannot support.

If a capability is unavailable, explicitly record:

```text
SUPPORTED
NOT SUPPORTED
SUPPORTED WITH LIMITATIONS
REQUIRES DIFFERENT API
UNKNOWN — MUST VERIFY
```

---

# 51.9 WASENDER DOCUMENTATION MUST BE TREATED AS AUTHORITATIVE

Before implementing any Wasender feature, inspect the official Wasender documentation supplied by the user.

In particular, verify:

- authentication
- PAT
- session API key
- session creation
- session lifecycle
- QR handling
- webhook registration
- webhook authentication
- incoming message events
- outgoing messages
- media
- delivery events
- message IDs
- errors
- rate limits
- retries
- session isolation

Never infer Wasender behavior from WAHA behavior.

---

# 51.10 PAT AND SESSION API KEY MUST BE REPRESENTED SEPARATELY

The configuration system must make the distinction impossible to misunderstand.

For example:

```text
WASENDER_PERSONAL_ACCESS_TOKEN
```

and:

```text
WASENDER_SESSION_API_KEY
```

must never be treated as interchangeable credentials.

The UI should clearly explain their scopes.

The database should never store a PAT as though it were a session credential.

The agent must test:

```text
PAT-only operations
Session-key operations
Invalid credential
Wrong session key
Expired/invalid credential
```

where supported.

---

# 51.11 MULTIPLE WASENDER SESSIONS

The architecture must support multiple WhatsApp sessions.

Example:

```text
Account
├── Session A → WhatsApp Number A
├── Session B → WhatsApp Number B
└── Session C → WhatsApp Number C
```

Every message, workflow, assignment, and event must remain associated with the correct session.

Test that:

```text
Session A cannot accidentally send using Session B.
```

This must be a dedicated automated test.

---

# 51.12 TWO-NUMBER REAL-WORLD TEST

After the simulated provider passes, perform a controlled real-world test using two WhatsApp numbers owned/controlled by the user.

### Number A

Acts as sender/customer.

### Number B

Acts as recipient/business.

Test:

```text
A → keyword → B
B → automated workflow → A
A → reply → B
```

Verify the entire chain.

Then reverse the roles.

Do not use uncontrolled third-party numbers for testing.

The test must verify:

- inbound message
- keyword detection
- variant assignment
- workflow execution
- text
- media
- delay
- reply
- attribution
- statistics
- database persistence

---

# 51.13 TEST THE WORKFLOW ENGINE WITHOUT WHATSAPP FIRST

The workflow engine must be independently testable.

Create a fake event:

```text
IncomingMessage {
    contact: test-contact
    keyword: TEST
}
```

Then execute:

```text
Workflow A
```

without WhatsApp.

The system must prove:

```text
trigger
→ assignment
→ workflow
→ delay
→ action
→ state transition
→ completion
```

before connecting a real WhatsApp number.

---

# 51.14 TIME MUST BE MOCKABLE

Never make automated tests wait for real delays.

If a workflow says:

```text
wait 120 seconds
```

the test must be able to simulate:

```text
+120 seconds
```

using:

- fake clock
- injected time provider
- deterministic scheduler
- equivalent testing mechanism

This is mandatory for scalable testing.

---

# 51.15 MEDIA MUST BE TESTABLE WITHOUT REAL PRODUCTION FILES

Create deterministic test fixtures:

```text
test-image.jpg
test-audio.mp3
test-video.mp4
test-document.pdf
```

The workflow tests must verify that the correct file reaches the correct workflow action.

For example:

```text
Variant A → audio-A.mp3

Variant B → audio-B.mp3

Variant C → audio-C.mp3
```

The system must prove that files never cross variants.

---

# 51.16 PROGRAMMATIC WORKFLOW CREATION MUST BE FIRST-CLASS

The programmatic workflow interface is not an optional convenience.

It is a core requirement.

The user must eventually be able to instruct an AI coding agent:

> "Create a workflow for Variant A using this text, this image, this audio, wait between 30 and 90 seconds, then send this video."

The agent should be able to interact with the CRM's API/database abstraction rather than manually clicking every node.

The generated workflow must then appear correctly in the visual builder.

---

# 51.17 BULK WORKFLOW GENERATION

The system must support programmatic creation of many variants.

Example:

```text
Variant A
Variant B
Variant C
...
Variant Z
...
Variant 210
```

Each may contain different:

- text
- image
- audio
- video
- delays
- conditions
- workflow paths

The system must not require manually constructing each workflow.

---

# 51.18 IMPORT/EXPORT WORKFLOW DEFINITIONS

Where practical, support a portable workflow representation.

Example:

```json
{
  "name": "Variant A",
  "trigger": {},
  "nodes": [],
  "edges": []
}
```

The exact schema must follow the actual implementation.

This allows:

```text
AI → generate JSON → validate → import workflow
```

and:

```text
CRM → export workflow → AI → modify → re-import
```

This should be considered strongly for V2 if not appropriate for V1.

---

# 51.19 HUMAN + AI WORKFLOW CREATION

The system should support two creation modes:

### Human

```text
Visual Builder
```

### AI/Programmatic

```text
API
```

They must converge into the same workflow engine.

Do not create an AI-specific workflow execution engine.

AI only creates/configures workflows.

---

# 51.20 AI MUST NOT BE REQUIRED FOR CORE AUTOMATION

The core system must work deterministically without an AI API.

AI is optional.

The following must work without AI:

```text
keyword
workflow
delay
media
branch
variant
statistics
reply attribution
pipeline
follow-up
```

Later:

```text
AI
 ↓
generate workflow
generate copy
personalize
analyze statistics
```

can be added without redesigning the core.

---

# 51.21 FOLLOW-UP CONTEXT

The system must preserve enough conversation history to determine:

```text
customer
conversation
campaign
variant
workflow
workflow step
message
reply
timestamp
```

A customer reply must be associated with the correct experiment context.

The system should not attempt to infer attribution using AI.

Use deterministic IDs and relationships.

---

# 51.22 CRM MUST BE THE SOURCE OF TRUTH

If context-aware automation is enabled, relevant WhatsApp interactions must flow through the CRM.

The agent must investigate whether DeskcommCRM's current UI allows the operator to:

```text
receive message
read conversation
reply from CRM
send media
continue conversation
```

If it does, preserve and connect that functionality to Wasender.

If it does not, identify the minimum UI/API changes required.

The goal is:

```text
WhatsApp
 ↓
Wasender
 ↓
Deskcomm
 ↓
CRM Inbox
 ↓
Human reply
 ↓
Wasender
 ↓
WhatsApp
```

The user should not be forced to switch to another device/application simply to maintain CRM history.

---

# 51.23 HUMAN REPLY MUST BE RECORDED

When a human responds from the CRM:

```text
CRM reply
 ↓
Wasender
 ↓
WhatsApp
```

the outgoing message must be persisted as part of the conversation.

The system must distinguish:

```text
automation-generated message
```

from:

```text
human-generated message
```

This is important for analytics and context.

---

# 51.24 FOLLOW-UP ENGINE MUST NOT CREATE DUPLICATES

If the customer replies:

```text
Customer replied
```

the system must evaluate whether active automated follow-ups should:

- continue
- pause
- stop
- transition

according to workflow rules.

Do not blindly continue sending messages after a customer has already responded.

This behavior must be explicitly configurable.

---

# 51.25 PIPELINE STAGE TRACKING

Workflow execution should be capable of updating the CRM's existing pipeline/stage representation.

Example:

```text
Keyword received
→ New lead

Workflow step 2
→ Contacted

Customer replies
→ Engaged

Customer requests price
→ Qualified

Purchase
→ Converted
```

Do not create a second independent CRM pipeline system if Deskcomm already has one.

---

# 51.26 STATISTICS MUST BE EVENT-BASED

Statistics should derive from persisted events rather than mutable counters alone.

Example:

```text
message_sent
variant_assigned
message_delivered
customer_replied
workflow_completed
conversion
```

Aggregated statistics can then be calculated safely.

This protects historical accuracy.

---

# 51.27 EXPERIMENT DATA MUST BE IMMUTABLE HISTORICALLY

If Variant A changes tomorrow, yesterday's customers must still be attributed to the version of Variant A they actually received.

Therefore consider versioning:

```text
Variant
   ↓
Variant Version
   ↓
Workflow Version
```

This can be introduced progressively if necessary.

At minimum, historical execution records must retain enough information to reconstruct what happened.

---

# 51.28 RANDOMIZATION MUST BE REPRODUCIBLE IN TESTS

Production:

```text
random
```

Testing:

```text
deterministic seed
```

This allows the QA system to reproduce a failing experiment assignment.

---

# 51.29 STATISTICAL CLAIMS MUST BE HONEST

Do not label a variant:

> "Winner"

simply because it currently has a higher reply rate.

The system should distinguish:

```text
raw performance
```

from:

```text
statistically meaningful result
```

Advanced statistical significance can be V3.

V1 should at least show sample size.

Example:

```text
Variant A
Replies: 12
Assigned: 20
Reply rate: 60%
Sample: 20
```

rather than pretending this is definitive proof that A is superior.

---

# 51.30 QA AGENT MUST TEST LIKE A HUMAN

The Playwright QA agent must not only call APIs.

It must actually interact with the application UI.

It must:

- log in
- navigate
- click
- drag
- drop
- type
- upload
- connect nodes
- save
- activate
- deactivate
- inspect statistics
- open conversations
- send messages
- inspect workflow state

The visual workflow builder must therefore be tested as a real user interface.

---

# 51.31 QA AGENT MUST ALSO TEST THE PROGRAMMATIC PATH

For every major visual feature, where applicable:

```text
UI creation
```

and:

```text
API creation
```

must be tested.

Then verify both result in equivalent persisted state.

---

# 51.32 FAILURE INJECTION

The automated test suite must intentionally simulate failures.

Examples:

```text
Wasender unavailable
Wasender timeout
invalid session key
duplicate webhook
malformed webhook
media upload failure
database transient failure
worker restart
Docker restart
workflow interrupted during wait
```

The system must recover safely.

---

# 51.33 OBSERVABILITY

Implement or preserve useful:

```text
structured logs
workflow execution logs
provider logs
webhook logs
error IDs
correlation IDs
```

A workflow execution should be traceable:

```text
Customer
→ Message
→ Event
→ Variant
→ Workflow
→ Node
→ Provider request
→ Provider response
```

This is essential so the user does not have to manually debug production failures.

---

# 51.34 NO SILENT FAILURES

If a workflow cannot execute:

```text
FAILED
```

must be persisted.

The system must provide enough information to determine:

- which workflow
- which customer
- which node
- which provider
- which error
- when
- retry status

---

# 51.35 RETRIES MUST BE CONTROLLED

Provider failures should not automatically create duplicate messages.

Implement idempotency-aware retry behavior.

For example:

```text
attempt 1
 ↓
timeout
 ↓
determine whether provider accepted request
 ↓
retry only when safe
```

Never blindly retry message-sending operations.

---

# 51.36 RATE LIMITING AND SAFETY

Do not build the system around aggressive message blasting.

Respect:

- Wasender limits
- WhatsApp policies
- reasonable delays
- user opt-outs
- stop conditions

The system must have safeguards against accidental runaway workflows.

Example:

```text
100 contacts
+
workflow bug
=
100,000 accidental messages
```

There should be execution limits / kill switches where appropriate.

---

# 51.37 GLOBAL WORKFLOW KILL SWITCH

Provide an administrative mechanism to stop active automation if a serious bug is detected.

For example:

```text
PAUSE ALL AUTOMATIONS
```

This should prevent new automated sends while preserving existing data.

This is highly recommended before real production use.

---

# 51.38 VARIANT KILL SWITCH

Each variant must independently support:

```text
ACTIVE
PAUSED
```

Pausing a variant prevents new assignments while preserving historical records.

---

# 51.39 SESSION KILL SWITCH

If a Wasender session behaves unexpectedly, administrators should be able to disable automation for that session without disabling the entire CRM.

Conceptually:

```text
Session A → automation enabled
Session B → automation paused
```

---

# 51.40 FINAL ACCEPTANCE TEST

Before declaring the system production-ready, run this complete scenario:

```text
Create two Wasender sessions

Session A
Session B

Create campaign

Create Variant A
Create Variant B
Create Variant C

Create three different workflows

Attach different text

Attach different image

Attach different audio

Attach different video

Configure delays

Activate all variants

Send keyword from controlled WhatsApp number

Verify assignment

Verify workflow execution

Verify media

Verify delays

Reply from customer

Verify attribution

Verify statistics

Pause Variant B

Send another keyword

Verify B receives no new assignment

Reactivate Variant B

Verify it can receive assignments

Pause entire automation system

Verify no automated messages are sent

Resume automation

Verify normal operation

Restart worker

Verify waiting workflows resume correctly

Restart Docker application

Verify state remains intact

Inspect database

Inspect logs

Inspect UI

Run Playwright regression suite
```

Only after this passes should the system be considered ready for real usage.

---

# 51.41 FINAL ENGINEERING PRINCIPLE

At every decision point, prefer:

```text
Existing Deskcomm functionality
        ↓
Reuse

Existing WA CRM workflow technology
        ↓
Adapt

Existing Wasender capability
        ↓
Integrate

Missing functionality
        ↓
Build minimally
```

Never:

```text
Existing functionality
        ↓
Rewrite
```

Never:

```text
Open-source workflow builder
        ↓
Rebuild from scratch
```

Never:

```text
WAHA behavior
        ↓
Assume Wasender behaves identically
```

Never:

```text
AI
        ↓
Make core automation dependent on it
```

And never:

```text
Feature request
        ↓
Implement everything
        ↓
Test at the end
```

The required discipline remains:

**inspect → isolate → implement minimally → test → verify → checkpoint → only then increase complexity.**

---

# you asked

message time: 2026-08-20 12:57:45

You have still forgotten something. Find it and add it.

---

# chatgpt response

You're right. The previous additions still missed a few requirements that are important to your actual operating model—especially **the distinction between the CRM's runtime AI and your external coding/agent layer**, the **existing Deskcomm dependencies that must be preserved**, and the **follow-up/content import system**.

Append this section to the bottom of the master prompt.

---

# 52. ADDITIONAL OPERATING-MODE REQUIREMENTS

These requirements are additive and mandatory. They specifically address the distinction between the existing Deskcomm infrastructure, the new Wasender/workflow system, and external AI coding agents.

---

## 52.1 DISTINGUISH THREE DIFFERENT TYPES OF "AI"

Do not confuse these three concepts.

### A. Deskcomm runtime AI

This is the AI functionality already present inside DeskcommCRM.

The repository currently documents:

- Vercel AI SDK
- Anthropic / Google / OpenAI providers
- AI Gateway
- OpenAI embeddings for RAG
- AI workers/features

The existing setup documentation confirms that AI provider credentials are used for AI features, while the application can start without configuring those AI features. ([GitHub](https://github.com/melgarafael/DeskcommCRM/blob/main/docs/SETUP.md?utm_source=chatgpt.com))

Therefore:

> **Do not remove Deskcomm's AI architecture. Do not require runtime AI for the V1 automation engine.**

---

### B. External coding/automation agent

The user may use:

```text
Claude Code
OpenCode
other coding/agent systems
```

to communicate with the CRM programmatically.

This is NOT the same thing as Deskcomm runtime AI.

The user should be able to use an external agent to:

```text
create workflow
modify workflow
create variants
upload/reference media
create follow-up sequences
activate/deactivate variants
read statistics
generate bulk workflow definitions
```

without requiring Deskcomm to have an AI API key.

---

### C. Future AI inside the CRM

Later, the user may configure:

```text
Anthropic
OpenAI
Google
other supported provider
```

for:

- AI-generated replies
- personalization
- summarization
- lead analysis
- copy generation
- intelligent classification
- RAG

These are future capabilities.

They must not become dependencies of the deterministic workflow engine.

---

# 52.2 V1 MUST WORK WITH ZERO RUNTIME AI API KEYS

The installation process must explicitly test:

```text
Anthropic key = absent
OpenAI key = absent
Google AI key = absent
```

where those keys are optional.

The application must still be able to:

```text
start
login
connect database
connect Wasender
receive webhook
process keyword
assign variant
execute workflow
send text
send media
wait
record reply
calculate statistics
```

If some unrelated Deskcomm feature requires AI, that feature may be unavailable/degraded.

But:

> **Core CRM + WhatsApp + automation + experimentation must not fail merely because runtime AI is not configured.**

---

# 52.3 CREATE AN "OPTIONAL DEPENDENCY MATRIX"

Before deployment, document every environment variable and classify it:

| Dependency | Required for app boot? | Required for CRM? | Required for WhatsApp? | Required for automation? | Optional V1? |
|---|---:|---:|---:|---:|---:|
| Supabase | | | | | |
| Supabase Auth | | | | | |
| Supabase Storage | | | | | |
| Upstash Redis | | | | | |
| Wasender PAT | | | | | |
| Wasender Session Key | | | | | |
| Anthropic | | | | | |
| OpenAI | | | | | |
| Google AI | | | | | |
| Sentry | | | | | |
| Resend | | | | | |
| Nuvemshop | | | | | |

The current Deskcomm documentation identifies Supabase, Upstash Redis, WAHA, AI providers, Sentry, Resend and other integrations. ([GitHub](https://github.com/melgarafael/DeskcommCRM/blob/main/docs/SETUP.md?utm_source=chatgpt.com))

The agent must verify the current source code rather than assuming the documentation is perfectly up to date.

---

# 52.4 DO NOT DELETE UNUSED INTEGRATIONS

If an integration is not required for V1, do not automatically remove it.

For example:

```text
AI
Sentry
Resend
Nuvemshop
```

may remain installed/configurable even if they are not needed for the initial automation system.

The goal is:

> **Disable unnecessary runtime requirements, not destroy existing capabilities.**

---

# 52.5 EXISTING STORAGE SHOULD BE REUSED FIRST

DeskcommCRM currently documents Supabase Storage as its WhatsApp media storage with a private media bucket. ([GitHub](https://github.com/melgarafael/DeskcommCRM/blob/main/README.md?utm_source=chatgpt.com))

Therefore the first implementation should use the existing storage abstraction.

Do not introduce Cloudflare R2 unless:

1. existing storage cannot satisfy the requirement;
2. scale testing demonstrates a need;
3. or the user explicitly requests R2.

If R2 is eventually introduced, use:

```text
StorageProvider
├── SupabaseStorage
└── CloudflareR2Storage
```

rather than coupling workflows directly to R2.

---

# 52.6 FOLLOW-UP CONTENT MUST BE SEPARATE FROM WORKFLOW LOGIC

Do not hardcode follow-up copy inside the workflow engine.

Separate:

```text
Workflow
```

from:

```text
Message Content
```

and:

```text
Media Asset
```

Conceptually:

```text
Workflow
   ↓
Step
   ↓
Message Template
   ↓
Text / Media
```

This allows the user to change copy without rebuilding the workflow.

---

# 52.7 SUPPORT IMPORTED FOLLOW-UP CONTENT

The system should support importing predefined follow-up content per stage.

For example:

```text
Stage 1
├── Message A
├── Message B
└── Message C

Stage 2
├── Message A
├── Audio A
└── Image A

Stage 3
├── Video A
└── Message A
```

The exact import format can be:

```text
CSV
JSON
```

or another format already compatible with the application.

Do not build a complicated content management system in V1 if a simple structured import is sufficient.

---

# 52.8 SUPPORT PERSONALIZATION WITHOUT RUNTIME AI

Message templates should support deterministic variables.

For example:

```text
{{first_name}}
{{phone}}
{{company}}
{{pipeline_stage}}
{{variant}}
{{agent_name}}
```

The workflow engine should resolve these variables from CRM data.

This allows personalized follow-ups without an AI API.

Example:

```text
Hello {{first_name}}, here is the information you requested...
```

The result is generated deterministically.

---

# 52.9 EXTERNAL AI AGENTS MAY GENERATE CONTENT, BUT THE CRM EXECUTES IT

The architecture should allow:

```text
Claude Code / OpenCode
        ↓
CRM API
        ↓
Create/update workflow
        ↓
CRM workflow engine
        ↓
Wasender
```

The external agent may generate:

- copy
- workflow definitions
- variant definitions
- personalization templates
- bulk campaign structures

But the CRM remains responsible for:

- validation
- authorization
- persistence
- scheduling
- execution
- attribution
- statistics

Never allow an external AI agent to bypass those controls by writing arbitrary database records.

---

# 52.10 MACHINE-READABLE API CONTRACT

The programmatic workflow interface must have a clearly documented API contract.

The external agent should be able to discover:

```text
available triggers
available actions
available conditions
available media types
workflow schema
variant schema
statistics schema
authentication requirements
```

Prefer an OpenAPI-compatible contract or another machine-readable format already used by Deskcomm.

This will make Claude Code/OpenCode integration much safer.

---

# 52.11 MCP SHOULD BE INVESTIGATED, NOT ASSUMED

WasenderAPI currently documents an MCP integration that can expose session management, messaging, contacts, and groups to MCP-compatible agents such as Claude Code and OpenCode. It requires a Personal Access Token and a paid Wasender plan according to the current documentation. ([WASenderApi](https://www.wasenderapi.com/llms.txt))

Do NOT automatically make Wasender MCP a dependency.

Instead investigate two possible architectures:

### Option A — Direct CRM API

```text
Claude Code
     ↓
Deskcomm API
     ↓
Workflow engine
     ↓
Wasender
```

### Option B — MCP

```text
Claude Code
     ↓
MCP
     ↓
Wasender
```

For workflow management and CRM consistency, prefer:

```text
Claude Code
     ↓
Deskcomm's authenticated API
     ↓
Deskcomm workflow engine
```

because otherwise an external agent could send messages through Wasender without the CRM knowing about them.

The CRM must remain the source of truth.

---

# 52.12 NEVER BYPASS THE CRM FOR PRODUCTION AUTOMATION

An external agent must NOT directly call:

```text
Wasender send-message
```

for production campaign execution if that message is supposed to belong to a Deskcomm workflow.

Instead:

```text
External Agent
      ↓
Deskcomm
      ↓
Workflow
      ↓
Variant
      ↓
Execution
      ↓
Wasender
```

This ensures statistics, attribution, history, and pipeline state remain correct.

Direct Wasender calls may be used for:

- development
- provider testing
- health checks
- diagnostic tests

but not as a replacement for the CRM execution engine.

---

# 52.13 WASENDER'S CAPABILITIES SHOULD BE LEVERAGED FULLY

Do not assume Wasender only supports text.

The current documentation exposes capabilities including:

- text
- image
- video
- audio
- document
- sticker
- contacts
- location
- polls
- quoted messages
- message information
- delivery/message logs
- session management
- groups
- presence
- contact operations

and media upload functionality. ([WASenderApi](https://www.wasenderapi.com/llms.txt))

Create a capability matrix and only expose actions in the workflow builder that are actually supported by the configured provider.

---

# 52.14 PROVIDER CAPABILITY DISCOVERY

The workflow builder should eventually be capable of determining:

```text
Wasender
   ↓
Supported actions
```

Then the UI should not display impossible actions.

For example, if a provider does not support an action:

```text
Send XYZ
```

the builder should mark it:

```text
Unavailable for current provider
```

rather than allowing a workflow that can never execute.

---

# 52.15 SESSION-SCOPED EXECUTION IS MANDATORY

Every workflow execution must know which WhatsApp session it belongs to.

Conceptually:

```text
Tenant
 ↓
WhatsApp Connection
 ↓
Session
 ↓
Campaign
 ↓
Variant
 ↓
Workflow Execution
 ↓
Message
```

Never infer the session from the contact alone.

A contact may interact through multiple WhatsApp numbers.

---

# 52.16 HUMAN AND AUTOMATED MESSAGES MUST SHARE ONE CONVERSATION

The CRM conversation should contain:

```text
incoming customer message
automated workflow message
human CRM response
customer response
automated follow-up
```

in one coherent chronological history.

Do not create a separate "automation conversation."

---

# 52.17 OUT-OF-BAND WHATSAPP ACTIVITY MUST BE HANDLED

If a user sends or receives messages outside the CRM UI but through the connected WhatsApp session, Wasender webhook events should synchronize those interactions into Deskcomm wherever technically possible.

The system should identify:

```text
incoming external message
outgoing external message
```

and distinguish them from:

```text
CRM-generated message
automation-generated message
```

This is critical for context-aware follow-up.

---

# 52.18 CRM REPLY MUST USE THE SAME WASENDER SESSION

When an agent replies from the Deskcomm inbox:

```text
Deskcomm
 ↓
Correct WhatsApp session
 ↓
Wasender
 ↓
Customer
```

Never accidentally send through another configured session.

This must be explicitly tested with two WhatsApp numbers.

---

# 52.19 SESSION CONNECTION UX

Because Wasender supports session creation, QR retrieval, connection, status, restart, disconnect, and other session operations, investigate whether Deskcomm can safely expose session management itself. ([WASenderApi](https://www.wasenderapi.com/llms.txt))

However:

### V1 recommendation

Prefer the simplest reliable connection flow.

If the Wasender dashboard can securely handle initial QR connection, allow that.

Then Deskcomm can import/manage the resulting session credentials.

Do not recreate QR infrastructure merely for visual consistency with WAHA.

### V2

Consider native QR/session management inside Deskcomm if there is a demonstrated UX benefit.

---

# 52.20 DO NOT CLAIM "ANTI-BAN" GUARANTEES

The system may implement:

- controlled delays
- random jitter
- rate limits
- working-hour restrictions
- stop conditions
- opt-out handling
- provider error handling

but QA must never claim:

> "The system guarantees the WhatsApp number will not be banned."

Testing should instead verify that the system respects configured sending controls and does not accidentally generate runaway traffic.

---

# 52.21 REAL-WORLD TESTING MUST START SMALL

The first real WhatsApp test must be extremely small:

```text
2 controlled numbers
1 keyword
1 campaign
2–3 variants
1–3 workflow steps
```

Only after this passes should the test increase complexity.

Never begin with:

```text
210 variants
1000 contacts
complex media workflows
```

on real WhatsApp.

---

# 52.22 FINAL SYSTEM BOUNDARIES

The final system must have these boundaries:

```text
┌──────────────────────────────────────────┐
│              CRM LAYER                   │
│ contacts / inbox / pipeline / history    │
└────────────────────┬─────────────────────┘
                     │
┌────────────────────▼─────────────────────┐
│          EXPERIMENT LAYER                │
│ variants / assignment / statistics       │
└────────────────────┬─────────────────────┘
                     │
┌────────────────────▼─────────────────────┐
│          WORKFLOW LAYER                  │
│ triggers / nodes / delays / branches     │
└────────────────────┬─────────────────────┘
                     │
┌────────────────────▼─────────────────────┐
│       CANONICAL PROVIDER LAYER           │
│ provider-independent actions/events      │
└────────────────────┬─────────────────────┘
                     │
┌────────────────────▼─────────────────────┐
│         WASENDER ADAPTER                 │
│ PAT / session key / webhooks / API       │
└────────────────────┬─────────────────────┘
                     │
┌────────────────────▼─────────────────────┐
│             WASENDER API                 │
└──────────────────────────────────────────┘
```

And separately:

```text
Claude Code / OpenCode
          │
          ▼
   Deskcomm API/MCP
          │
          ▼
Workflow / Experiment / Content APIs
```

while:

```text
Deskcomm Runtime AI
```

remains an optional independent subsystem.

---

## 52.23 NON-NEGOTIABLE FINAL TEST

Before production, prove all of the following simultaneously:

```text
AI API keys absent
        +
Deskcomm boots
        +
Supabase works
        +
existing CRM works
        +
Wasender works
        +
two sessions work
        +
keyword works
        +
workflow works
        +
media works
        +
variant assignment works
        +
statistics work
        +
reply attribution works
        +
CRM human reply works
        +
programmatic workflow creation works
        +
visual workflow creation works
        +
both produce equivalent workflows
        +
Docker restart works
        +
workflow recovery works
        +
existing Deskcomm functionality still passes
```

Only then move from V1 to V2.

**The objective is not merely to integrate Wasender into Deskcomm. The objective is to create a provider-independent automation and experimentation layer that sits cleanly on top of Deskcomm's existing CRM architecture, while preserving the ability to use external AI coding agents without making runtime AI a dependency.**

---

# you asked

message time: 2026-08-20 13:00:56

Okay, now also add a requirement so that if a new version of Deskcom CRM is available, we can pull out the version and rebuild our own software without rebuilding from scratch. That is mandatory and one of the most important things.

---

# chatgpt response

Absolutely. This is important enough that I would make it a **top-level architectural requirement**, not just a maintenance note.

Append this section to the master prompt:

---

# 53. UPSTREAM UPDATEABILITY / FORK-SAFE ARCHITECTURE — MANDATORY

This is one of the **highest-priority requirements of the entire project**.

We are building a customized version of DeskcommCRM, but we must **not create a dead-end fork**.

The project must remain capable of incorporating future upstream DeskcommCRM releases without rebuilding our customized CRM from zero.

The fundamental principle is:

> **DeskcommCRM remains the upstream foundation. Our custom functionality must remain as isolated and portable as reasonably possible.**

---

## 53.1 TRACK UPSTREAM DESKCOMMCRM

The repository must preserve a clear relationship with the original DeskcommCRM repository.

Maintain:

```text
upstream → original DeskcommCRM
origin   → our customized repository
```

The Git configuration should make this relationship explicit.

The agent must be able to determine:

```text
Current upstream version
Current customized version
Commits introduced by us
Files modified by us
Files inherited unchanged
```

---

# 53.2 NEVER LOSE THE UPSTREAM HISTORY

Do not flatten the Git history.

Do not create a completely new repository by copying the source files and deleting the original Git history.

The Git history must allow:

```text
git fetch upstream
```

followed by analysis of upstream changes.

---

# 53.3 CUSTOM CODE MUST BE IDENTIFIABLE

Every significant customization must be identifiable.

Prefer architectural separation such as:

```text
src/
├── existing-deskcomm/
└── custom/
    ├── wasender/
    ├── automation/
    ├── experiments/
    ├── workflow/
    ├── media/
    └── integrations/
```

The exact directories must follow the actual Deskcomm architecture.

Do **not** force this exact structure if it conflicts with the existing project.

The principle is what matters:

> **A future developer must be able to distinguish upstream code from custom code.**

---

# 53.4 MINIMIZE MODIFICATIONS TO UPSTREAM FILES

When implementing a feature, prefer:

### First choice

Add a new module.

### Second choice

Use an existing extension/interface/hook.

### Third choice

Create an adapter.

### Fourth choice

Modify an existing component minimally.

### Last resort

Deeply modify upstream code.

Before modifying an upstream file, ask:

> Can this functionality be implemented as an extension instead?

If yes, use the extension.

---

# 53.5 MAINTAIN AN UPSTREAM MODIFICATION REGISTRY

Create a machine-readable and human-readable record such as:

```text
docs/UPSTREAM-MODIFICATIONS.md
```

For every modified upstream file record:

```text
File:
Upstream responsibility:
Why we modified it:
Our modification:
Dependency on custom functionality:
How difficult to rebase:
Possible future extraction:
```

Example:

```text
src/whatsapp/...
Reason:
Replace WAHA provider with provider abstraction.

Risk:
Medium.

Future strategy:
Move provider-specific code into adapter.
```

---

# 53.6 CREATE AN UPSTREAM DIFF REPORT

The project must have a documented way to generate:

```text
UPSTREAM
   vs
CUSTOM
```

differences.

The report should identify:

```text
added files
deleted files
modified files
renamed files
database migrations
dependency changes
environment variables
Docker changes
Coolify changes
```

This report must be reproducible.

---

# 53.7 UPSTREAM UPDATE WORKFLOW

When a new DeskcommCRM release appears, the agent must NOT immediately merge it.

Instead execute:

```text
1. Detect new upstream version
2. Fetch upstream
3. Create update branch
4. Compare versions
5. Analyze upstream changes
6. Identify conflicts with custom modules
7. Update dependencies if required
8. Apply upstream changes
9. Reapply/retain custom functionality
10. Run database migration analysis
11. Run automated tests
12. Run integration tests
13. Run Playwright UI tests
14. Run Wasender tests
15. Run workflow tests
16. Run regression tests
17. Produce upgrade report
18. Human approval
19. Merge
```

No automatic production upgrade.

---

# 53.8 NEVER "REBUILD FROM SCRATCH" FOR AN UPSTREAM UPDATE

This requirement is absolute.

If DeskcommCRM releases:

```text
v1
→ v2
```

we must be able to produce:

```text
Deskcomm v2
+
our custom modules
=
our customized v2
```

without manually rebuilding the entire project.

The agent should use Git merge/rebase/cherry-pick/patch strategies as appropriate.

---

# 53.9 CREATE A CUSTOM PATCH/EXTENSION LAYER

Where technically appropriate, isolate custom modifications into:

```text
custom/
```

or equivalent extension boundaries.

The objective is to make the custom layer portable:

```text
Deskcomm vX
     +
Custom Layer
     ↓
Customized Deskcomm vX
```

Then:

```text
Deskcomm vX+1
     +
Custom Layer
     ↓
Customized Deskcomm vX+1
```

This is the desired long-term architecture.

---

# 53.10 DATABASE MIGRATIONS MUST ALSO BE UPSTREAM-SAFE

This requirement applies to the database, not only source code.

Maintain a clear distinction between:

```text
Upstream migrations
```

and:

```text
Custom migrations
```

Never edit an existing upstream migration after it has been applied.

Instead create a new migration.

Example:

```text
upstream:
001_initial.sql
002_contacts.sql
003_messages.sql

custom:
100_wasender.sql
101_workflows.sql
102_variants.sql
103_experiments.sql
```

The actual numbering must follow the project's migration system.

---

# 53.11 UPSTREAM DATABASE CHANGES MUST BE ANALYZED

When a new Deskcomm version changes the database:

```text
New upstream schema
        ↓
Compare with custom schema
        ↓
Detect conflicts
        ↓
Generate migration plan
        ↓
Test on database copy
        ↓
Run regression tests
```

Never blindly run upstream migrations against production.

---

# 53.12 CUSTOM DATABASE OBJECTS MUST HAVE CLEAR OWNERSHIP

Custom tables should have unmistakable ownership.

For example:

```text
automation_workflows
automation_nodes
automation_executions
experiment_variants
experiment_assignments
experiment_events
media_assets
```

The exact schema must be derived from the actual implementation.

Do not modify existing Deskcomm tables simply because creating a custom table is slightly more work.

---

# 53.13 CUSTOM API CONTRACT MUST REMAIN STABLE

The custom workflow API should be treated as its own compatibility layer.

For example:

```text
/api/automation/*
/api/experiments/*
/api/wasender/*
```

If upstream changes internal Deskcomm APIs, our custom API should remain stable wherever possible.

This protects:

```text
Claude Code
OpenCode
automation scripts
workflow importers
external integrations
```

from upstream changes.

---

# 53.14 CREATE AN UPSTREAM COMPATIBILITY TEST SUITE

The custom layer must have tests specifically designed to detect upstream breakage.

Example:

```text
Upstream update
       ↓
Custom compatibility tests
       ↓
PASS / FAIL
```

Test:

- CRM authentication
- contacts
- conversations
- messaging
- database access
- storage
- workflow creation
- workflow execution
- Wasender adapter
- variants
- statistics
- programmatic workflow API
- UI workflow builder

---

# 53.15 UPSTREAM UPDATE SIMULATION

Before declaring the architecture complete, perform an actual exercise:

1. Start from the current customized Deskcomm.
2. Identify the current upstream commit/version.
3. Create a simulated upstream update.
4. Apply upstream changes.
5. Reconstruct the customized application.
6. Verify that our custom functionality survives.
7. Measure how much manual intervention was required.

The objective is to prove—not merely assume—that the architecture is updateable.

---

# 53.16 "REBASE TEST" IS A RELEASE GATE

A release cannot be considered architecturally complete until the agent demonstrates that the custom layer can be reapplied to a newer upstream revision without reconstructing the application from scratch.

The report must state:

```text
Upstream version:
Our version:

Files changed upstream:
Files changed by us:

Conflicts:
Automatic resolutions:
Manual resolutions:

Database conflicts:
Dependency conflicts:

Custom functionality preserved:
YES / NO

Full rebuild required:
NO
```

If a full rebuild is required, the architecture has failed this requirement and must be improved before proceeding.

---

# 53.17 UPSTREAM UPDATE SHOULD NEVER DESTROY CUSTOM DATA

An upstream update must preserve:

```text
contacts
conversations
messages
media
pipelines
workflows
variants
experiments
statistics
session mappings
configuration
```

unless a deliberate migration explicitly changes them.

The agent must verify data integrity after every upgrade.

---

# 53.18 VERSION MATRIX

Maintain a compatibility matrix:

| Deskcomm Version | Custom Version | Wasender Adapter | DB Schema | Status |
|---|---|---|---|---|
| vX | Custom v1 | v1 | v1 | ✅ |
| vX+1 | Custom v1 | v1 | v2 | 🧪 |
| vX+2 | Custom v2 | v2 | v3 | Planned |

This becomes the project's upgrade map.

---

# 53.19 UPSTREAM RELEASE MONITORING

The project should make it easy for the coding agent to determine:

> "Is there a newer DeskcommCRM version?"

The agent should be able to inspect:

```text
upstream GitHub
tags
releases
commits
```

and report:

```text
Current:
Latest upstream:
Changes:
Potential conflicts:
Recommended upgrade:
```

Do not automatically upgrade production.

---

# 53.20 COOLIFY MUST ALSO BE UPDATE-SAFE

Our Coolify deployment must not depend on a fragile one-off manual configuration.

Document:

```text
Dockerfile
docker-compose configuration if applicable
environment variables
persistent volumes
database configuration
storage
network configuration
webhooks
domains
health checks
worker configuration
```

The application must be reproducible on Coolify.

If the source code is rebuilt from:

```text
new Deskcomm version
+
custom layer
```

the resulting application must be deployable through the same Coolify strategy.

---

# 53.21 DO NOT COUPLE CUSTOM FEATURES TO COOLIFY

Coolify is the deployment platform.

It must not become part of the application architecture.

The application should remain portable to:

```text
Coolify
Docker
VPS
other compatible infrastructure
```

where practical.

---

# 53.22 FINAL ARCHITECTURAL OBJECTIVE

The final architecture should conceptually look like:

```text
                UPSTREAM
            DeskcommCRM
                 │
                 │ Git
                 ▼
        ┌─────────────────┐
        │ Custom Extension│
        │      Layer      │
        ├─────────────────┤
        │ Wasender Adapter│
        │ Automation      │
        │ Workflow Engine │
        │ A/B Testing     │
        │ Variant Engine  │
        │ Statistics      │
        │ Media           │
        │ Programmatic API│
        └────────┬────────┘
                 │
                 ▼
       Customized Deskcomm
                 │
                 ▼
              Docker
                 │
                 ▼
              Coolify
```

When upstream releases a new version:

```text
             NEW DESKCOMM
                  │
                  ▼
          Analyze upstream
                  │
                  ▼
       Merge/rebase upstream
                  │
                  ▼
        Reapply custom layer
                  │
                  ▼
        Migration compatibility
                  │
                  ▼
          Full QA / E2E
                  │
                  ▼
        Customized NEW version
```

### FINAL RULE

> **The custom system must be designed as a maintainable downstream distribution of DeskcommCRM—not as a completely independent rewrite that merely started from DeskcommCRM.**

The success criterion is therefore not only:

**"Does the customized CRM work today?"**

It is:

**"Can we take tomorrow's DeskcommCRM, apply our custom layer, migrate our data safely, run the complete QA suite, and obtain our customized CRM again without starting over?"**

That requirement should be treated as a **release-blocking architectural constraint**, equal in importance to the Wasender integration and the workflow/variant engine.
