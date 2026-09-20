# Bookback — Firestore schemas

Multi-tenant. Path style: `orgs/{orgId}/...`

Beachhead examples use plumbing (RapidFlow Plumbing) but schema is vertical-agnostic.

---

## Collection: `orgs`

Workspace / tenant for one local service business.

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | e.g. "RapidFlow Plumbing" |
| `slug` | string | yes | unique, URL-safe |
| `industry` | string | no | e.g. `plumbing` |
| `timezone` | string | yes | e.g. `America/Denver` |
| `truckCount` | number | no | ICP signal (1–10) |
| `status` | string | yes | `trial` \| `active` \| `paused` \| `churned` |
| `createdAt` | timestamp | yes | |
| `updatedAt` | timestamp | yes | |

---

## Collection: `orgs/{orgId}/members`

| Field | Type | Required | Notes |
|---|---|---|---|
| `userId` | string | yes | Clerk user id |
| `email` | string | yes | |
| `displayName` | string | no | e.g. "Mike Torres" |
| `role` | string | yes | `owner` \| `admin` \| `dispatcher` \| `tech` |
| `createdAt` | timestamp | yes | |

---

## Collection: `orgs/{orgId}/businessConfig` (single doc `main`)

Hours, services, area, booking, handoff.

| Field | Type | Required | Notes |
|---|---|---|---|
| `hours` | map | yes | weekday → `{ open, close, closed }` |
| `afterHoursMode` | string | no | `off` \| `on_call` \| `handoff` |
| `services` | array of map | yes | `{ id, label, active, emergency? }` |
| `serviceArea` | map | yes | `{ zips: string[], cities?: string[], radiusMiles? }` |
| `bookingRules` | map | yes | slot length, buffers, same-day cutoff |
| `faqs` | array of map | no | `{ q, a }` for AI SMS |
| `handoff` | map | no | rules → notify member / stop AI |
| `phoneDisplay` | string | no | customer-facing number |
| `updatedAt` | timestamp | yes | |

### Example `services` entries (plumbing)

```json
[
  { "id": "leak_sink", "label": "Leak under sink", "active": true },
  { "id": "clogged_drain", "label": "Clogged drain", "active": true },
  { "id": "water_heater", "label": "Water heater", "active": true },
  { "id": "slab_leak", "label": "Slab leak", "active": true },
  { "id": "burst_pipe", "label": "Emergency burst pipe", "active": true, "emergency": true }
]
```

---

## Collection: `orgs/{orgId}/leads`

One doc per recovered missed-call / inbound SMS lead.

| Field | Type | Required | Notes |
|---|---|---|---|
| `source` | string | yes | `missed_call` \| `inbound_sms` \| `manual` |
| `status` | string | yes | `new` \| `qualifying` \| `ready_to_book` \| `booked` \| `handoff` \| `lost` \| `spam` |
| `callerName` | string | no | |
| `callerPhone` | string | yes | E.164 |
| `address` | map | no | `{ line1, city, zip }` |
| `issueSummary` | string | no | e.g. "Leak under kitchen sink" |
| `serviceId` | string | no | ref into businessConfig.services |
| `urgency` | string | no | `standard` \| `same_day` \| `emergency` |
| `qualification` | map | no | AI-collected answers |
| `missedCallAt` | timestamp | no | |
| `lastMessageAt` | timestamp | no | |
| `appointmentId` | string | no | set when booked |
| `createdAt` | timestamp | yes | |
| `updatedAt` | timestamp | yes | |

---

## Collection: `orgs/{orgId}/conversations`

| Field | Type | Required | Notes |
|---|---|---|---|
| `leadId` | string | yes | |
| `channel` | string | yes | `sms` |
| `status` | string | yes | `open` \| `closed` |
| `aiEnabled` | boolean | yes | false after handoff |
| `twilioConversationSid` | string | no | id only, no secrets |
| `createdAt` | timestamp | yes | |
| `updatedAt` | timestamp | yes | |

---

## Collection: `orgs/{orgId}/conversations/{conversationId}/messages`

| Field | Type | Required | Notes |
|---|---|---|---|
| `direction` | string | yes | `in` \| `out` |
| `body` | string | yes | |
| `sender` | string | yes | `lead` \| `ai` \| `human` |
| `twilioMessageSid` | string | no | id only |
| `createdAt` | timestamp | yes | |

---

## Collection: `orgs/{orgId}/appointments`

| Field | Type | Required | Notes |
|---|---|---|---|
| `leadId` | string | yes | |
| `serviceId` | string | no | |
| `title` | string | yes | e.g. "Kitchen sink leak — Maria R." |
| `startsAt` | timestamp | yes | |
| `endsAt` | timestamp | yes | |
| `status` | string | yes | `scheduled` \| `confirmed` \| `canceled` \| `completed` \| `no_show` |
| `address` | map | no | |
| `techUserId` | string | no | member id |
| `external` | map | no | Jobber / HCP / Google event ids |
| `bookedVia` | string | yes | `bookback_ai` \| `bookback_human` \| `imported` |
| `createdAt` | timestamp | yes | |
| `updatedAt` | timestamp | yes | |

---

## Collection: `orgs/{orgId}/integrations` (docs by provider)

Ids only — **no secrets** (secrets in Secret Manager / env).

| Field | Type | Required | Notes |
|---|---|---|---|
| `provider` | string | yes | `twilio` \| `google_calendar` \| `jobber` \| `housecall_pro` \| `stripe` |
| `accountId` | string | no | external account / workspace id |
| `phoneNumberSid` | string | no | Twilio |
| `calendarId` | string | no | |
| `customerId` | string | no | Stripe customer id |
| `subscriptionId` | string | no | Stripe sub id |
| `status` | string | yes | `connected` \| `disconnected` \| `error` |
| `updatedAt` | timestamp | yes | |

---

## Collection: `orgs/{orgId}/usage` (daily rollups) + `aiCostEvents`

### `usage/{yyyy-mm-dd}`

| Field | Type | Required | Notes |
|---|---|---|---|
| `smsSent` | number | yes | |
| `smsReceived` | number | yes | |
| `leadsCreated` | number | yes | |
| `appointmentsBooked` | number | yes | |
| `aiTokensIn` | number | no | |
| `aiTokensOut` | number | no | |
| `aiCostUsd` | number | no | |

### `aiCostEvents/{eventId}`

| Field | Type | Required | Notes |
|---|---|---|---|
| `leadId` | string | no | |
| `conversationId` | string | no | |
| `model` | string | yes | |
| `tokensIn` | number | yes | |
| `tokensOut` | number | yes | |
| `costUsd` | number | yes | |
| `createdAt` | timestamp | yes | |

---

## Collection: `orgs/{orgId}/billing` (doc `main`) — stub

| Field | Type | Required | Notes |
|---|---|---|---|
| `plan` | string | yes | `trial` \| `starter` \| `pro` |
| `stripeCustomerId` | string | no | id only |
| `stripeSubscriptionId` | string | no | id only |
| `status` | string | yes | `trialing` \| `active` \| `past_due` \| `canceled` |
| `currentPeriodEnd` | timestamp | no | |
| `updatedAt` | timestamp | yes | |

---

## Example lead document (plumbing)

```json
{
  "source": "missed_call",
  "status": "ready_to_book",
  "callerName": "Maria R.",
  "callerPhone": "+13035550142",
  "address": { "line1": "1842 Lowell Blvd", "city": "Denver", "zip": "80211" },
  "issueSummary": "Leak under kitchen sink",
  "serviceId": "leak_sink",
  "urgency": "same_day",
  "qualification": {
    "waterShutOff": true,
    "location": "kitchen sink cabinet"
  },
  "missedCallAt": "2026-09-20T20:12:00Z",
  "lastMessageAt": "2026-09-20T20:18:00Z",
  "createdAt": "2026-09-20T20:12:30Z",
  "updatedAt": "2026-09-20T20:18:00Z"
}
```

---

## What powers each screen

- **Landing / Sign in** → marketing + Clerk auth (no Firestore yet)
- **Onboarding** → creates `orgs`, `members`, `businessConfig`, `integrations`
- **Dashboard KPIs + recent leads** → `leads` (+ daily `usage` rollups)
- **Leads inbox / SMS thread** → `leads`, `conversations`, `messages`
- **Book appointment CTA** → writes `appointments`, updates lead status
- **Calendar** → `appointments` (+ Jobber/HCP/Google ids on `integrations` / `external`)
- **Settings** → `businessConfig`, `billing` stub
- **Admin tenants / usage** → cross-org `orgs` + `usage` / `aiCostEvents`
