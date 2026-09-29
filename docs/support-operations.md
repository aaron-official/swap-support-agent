# Swap support agent: support operations and design

Swap Africa is run by one person. As sellers, buyers and riders grow, support
will take most of that person's day. This document works out what support at
Swap actually involves, based on the platform code in `aaron-official/swap` as
of 2026-09-29, and how an agent built on the sales agent
(`aaron-official/swap-sales-agent`) should handle each part.

The goal:

- The agent answers most questions by itself, from facts it can check.
- For everything else, it collects the facts and hands the owner a short brief
  with a recommended action.
- The owner makes decisions and nothing else.
- The agent never moves money, never makes account changes, and never decides
  a dispute.

Contents:

1. [Who contacts support, and how we know who they are](#1-who-contacts-support-and-how-we-know-who-they-are)
2. [The support catalogue](#2-the-support-catalogue)
3. [What the agent may do](#3-what-the-agent-may-do)
4. [Identity checks](#4-identity-checks)
5. [How the agent gets facts](#5-how-the-agent-gets-facts)
6. [Cases, priorities and response times](#6-cases-priorities-and-response-times)
7. [Proactive support](#7-proactive-support)
8. [Channels and hours](#8-channels-and-hours)
9. [The AI model and customer data](#9-the-ai-model-and-customer-data)
10. [What to reuse from the sales agent](#10-what-to-reuse-from-the-sales-agent)
11. [Gaps in the Swap platform that support will hit](#11-gaps-in-the-swap-platform-that-support-will-hit)
12. [Decisions for the owner](#12-decisions-for-the-owner)
13. [Build plan](#13-build-plan)

---

## 1. Who contacts support, and how we know who they are

| Who | Has a Swap account? | How we recognise them | What they usually want |
|---|---|---|---|
| **Buyer** | No. Guest session on a link, valid 24 hours; the link can be reopened | The Mobile Money number they paid with (`commerce.transactions.buyer_msisdn`) and the link code in their URL (`swapafrica.online/t/<code>`) | "I paid, where is my item?", "Where is my refund?", "Is this link real?" |
| **Seller** | Yes (`identity.profiles`, role `seller`) | Their registered phone, which is SMS-verified and locked | Sign-in, KYC, wallet and withdrawals, PIN locks, fees, a deal that is stuck |
| **Rider** | Yes (role `rider`) | Registered phone | Application status, activation, gigs, earnings, withdrawals |
| **Contract buyer / service seller** | Buyer: guest; seller: account | Contract link code, phone | Milestone funding, review, auto-release, disputes |
| **Everyone else** | No | Nothing | How Swap works, scam reports, press, law enforcement, data requests, "where did you get my number?" (from the sales agent) |

People currently reach Swap by:

- `mailto:support@swapafrica.online`: the seller account page, the PIN and
  withdrawal lock screens, and the rider "access inactive" screen.
- The contact form on `swapafrica.online`, which sends a Pushover
  notification to the owner's phone (`apps/buyer/app/api/contact/route.ts`).

There is no WhatsApp support number yet. Most Ugandans will expect one.

## 2. The support catalogue

Every situation below comes from a real state, error or message in the code.

Each table has four columns:

- **Checks:** what the agent looks up before answering.
- **Agent does:** what it handles by itself.
- **Owner:** when it goes to you.

A reference to *G1* and similar points at a platform gap in
[section 11](#11-gaps-in-the-swap-platform-that-support-will-hit).

### 2.1 Buyers: goods deals

The deal states are `PENDING_FUNDING`, `FUNDED`, `RIDER_ASSIGNED`,
`AT_PICKUP`, `VERIFYING`, `IN_TRANSIT`, `AT_DROPOFF`, `DELIVERED`,
`COMPLETED`, `VOIDED` and `DISPUTED` (`domain/transactions/state_machine.rs`).

| Situation | Checks | Agent does | Owner |
|---|---|---|---|
| "The payment prompt never came" | Deal is `PENDING_FUNDING`; a payment exists, or doesn't | Explains the usual causes: wrong number typed, the network, not enough balance, the prompt timing out. Tells them to try again from the link. | No |
| "I paid but it still says pending" | Payment status (`pending` / `indeterminate`), minutes since the prompt | Under ~10 minutes: the payment is being confirmed (the reconciliation worker checks every 60s). Longer: opens a P1 case. | Yes, after 30 minutes (a stuck payment) |
| "The link says expired" | Link `expired`; deal voided as `stale_unfunded_link` after 24 hours | Unpaid links expire after 24 hours and nothing was taken. Ask the seller for a new link. | No |
| "Money left my phone but the link expired" | Payment records for that number | Rare, and serious | Yes, P1 |
| "No rider yet" | `FUNDED` since when; `gig_offers` history | Explains that riders are being offered the job. After 30 minutes, opens a case. | Yes. There is no way to cancel and refund a funded deal (G1) |
| "The seller says the item is sold out" or "I want to cancel before pickup" | Deal state `FUNDED` / `RIDER_ASSIGNED` / `AT_PICKUP` | Collects both sides | Yes. Needs a void and refund that doesn't exist yet (G1) |
| "What are these photos?" | `VERIFYING` | Explains: approve and the rider brings it; reject and you get a full refund and the item stays with the seller | No |
| "Where is the rider?" | `IN_TRANSIT` / `AT_DROPOFF`, gig status, time since last change | Gives the status. Never shares the rider's live location or number beyond what the link page shows. | Yes, if nothing has moved for over an hour |
| "The QR code won't scan" | `AT_DROPOFF` | The rider can type the 6-digit code shown on the buyer's page instead | No |
| "I refused at the door; why didn't I get the delivery fee back?" | `VOIDED`, `void_reason = buyer_refused`, who pays delivery | Explains: a refusal refunds the item price; the delivery fee pays the rider for the trip. The terms say something different (G6). | Only if they dispute it |
| "Where is my refund?" | Refund journal and payment: reserved, sent, settled or failed | Settled: gives the time and the number it went to (masked as `07XX XXX 123`). Sent: normal delay. Failed: see the next row. | Yes, if failed |
| Refund failed | Audit event `refund_failed_alert` | Tells them it's being handled | Yes, P1. Only an audit event records it: no retry, no admin screen, no message to the buyer (G2) |
| "I opened a dispute; what now?" | `DISPUTED`, statements and evidence present | Explains the process, reminds them to add their statement and photos on the link page, and asks for missing facts | The decision is always yours |
| "Is this Swap link real?" | Domain is `swapafrica.online`; link code exists; seller name matches | Confirms or warns. Very useful in a market full of fake payment links. | Yes, if fake: a scam report |
| "The seller wants me to pay outside Swap" | None | Warns them never to pay outside the link | Yes: a seller conduct report |

### 2.2 Buyers and sellers: service contracts

Contract states run from `proposed` / `countered` through `accepted`,
`funded`, `completed` and `voided`. Milestone states are `awaiting_funding`,
`locked`, `in_review`, `released`, `auto_released`, `changes_requested` and
`disputed` (`docs/architecture/05-domain-flows.md`).

| Situation | Agent does | Owner |
|---|---|---|
| "How do milestones work?" / "Why did the seller get the deposit already?" | Explains: deposits release at once; later milestones are funded one by one and released on approval | No |
| "The milestone released without my approval" | Checks `auto_released`: the review window lapsed. Explains the rule. | Yes, if they dispute it |
| "The buyer won't fund the next milestone" | Explains that there is no deadline: send a reminder, or void | No |
| A milestone dispute | Same as for goods disputes | Yes, always |
| Negotiation limits | Explains the six-version cap and that milestones must add up to the total | No |

### 2.3 Sellers: accounts and sign-in

| Situation | Checks | Agent does | Owner |
|---|---|---|---|
| "The sign-up code never came" | SMS delivery records; SMS credit low (`credit_balance_probe`) | Wait 60s and resend; check the number format | Yes, if SMS credit is out (it affects everyone) |
| "This number is already registered" (`409 phone_in_use`) | A profile with that phone exists | Tells them to sign in or reset the password | Yes, if they say it isn't theirs (possible fraud) |
| "This number can't be registered" | `identity.deleted_accounts`: a 12-month block or a fraud flag | Explains the block without the reason | Yes, if they appeal |
| "Forgot password" | None | Walks them through the SMS reset (`/signin/recover`), which is self-service | No |
| "I changed my SIM / number" | Profile phone | Explains that only Swap can change it, on a call, with a code sent to the new number | Yes: you do the audited admin action (`/v1/admin/customers/{id}/phone/*`) |
| Add an email, Google sign-in | Contacts | Guides them | No |
| Delete the account | None | Points to Account > Delete account. Explains what is kept (financial records, 7 years). | No |
| Suspended account | `suspended`, `suspended_reason` | Says it is suspended and that you will review | Yes |

### 2.4 Sellers: identity verification (KYC)

| Situation | Agent does | Owner |
|---|---|---|
| "How do I verify?" | Didit steps. Manual photo upload is only the fallback. | No |
| Didit failed or keeps failing | Tips: light, a real ID, no glare. Offers the manual fallback in the app. | If it fails repeatedly |
| "Verification pending for days" | Checks KYC status and age | Yes, if manual review is waiting on you |
| "Why was I rejected?" | Gives the recorded reason in plain words | Appeals |
| "Can I send my ID on WhatsApp?" | **No.** IDs go only through the app, into private storage. | No |

### 2.5 Sellers and riders: wallet and withdrawals

Facts from `domain/withdrawals.rs` and `http/wallet.rs`:

- **Limits:** Mobile Money withdrawals are UGX 500 to 7,000,000; bank
  withdrawals start at UGX 50,000. Fees come from a band table.
- **KYC:** a seller's first withdrawal needs verified KYC.
- **Name check:** the first withdrawal to a new number must pass a name check
  (`MsisdnUnverified`).
- **Failed withdrawals** are reversed to the wallet automatically, with a
  "Withdrawal failed" notice.
- **PIN locks:** three wrong PINs lock withdrawals for 5 hours. Repeated
  cycles escalate, and at the top level the lock is indefinite until support
  resets the PIN.

| Situation | Agent does | Owner |
|---|---|---|
| "The withdrawal failed" | Confirms the money is back in the wallet and gives the likely cause (network, name check, provider) | If it keeps failing |
| "The withdrawal is pending" | Checks it is `sent` and for how long | Yes, P1 if it has been in transit over 2 hours |
| "Recipient could not be verified" | The number must be registered in the account holder's own name with MTN or Airtel | No |
| "Insufficient balance" or fee questions | Shows the fee for the amount from the band table | No |
| "Withdrawals locked for 5 hours" | Says when it unlocks | No |
| "Locked indefinitely" | Starts the identity check (section 4) | Yes: the PIN reset is yours (`/v1/admin/sellers/{id}/reset-pin`) |
| "Swap owes me money" or a balance dispute | Pulls the wallet history | Yes |

### 2.6 Sellers: day-to-day use

The agent answers these from the knowledge base with no escalation:

- creating links, service contracts and photos;
- how buyers pay;
- delivery pricing (UGX 2,000 plus 700 per km, minimum 3,500);
- commission;
- who pays delivery;
- notifications;
- the logo;
- "Rider Wange", the seller's own rider.

### 2.7 Riders

Rider states are `not_submitted`, `didit_pending`, `didit_approved`,
`under_review`, `approved` and `rejected`. Incomplete accounts are deleted
after 5 days (`docs/architecture/09-rider-vetting-and-onboarding.md`,
`worker/mod.rs`).

| Situation | Agent does | Owner |
|---|---|---|
| "I applied; what now?" | Gives the application status and the next step | Reviews and provisioning are yours |
| "I didn't get my password or code" | Never sends passwords. Explains activation by code, or that the SMS goes to the applied number. | Yes, to resend through the admin app |
| "My account disappeared" | Explains the 5-day expiry and asks them to reapply | No |
| No gigs, or the offer expired | Explains that offers last about 45 seconds and go to the nearest online rider | No |
| Trouble at pickup or drop-off | Gives the steps: arrive, three photos, scan or code | Yes, if a deal is stuck |
| The wait fare is wrong | The on-screen meter is cosmetic; wait pay is computed on the server at the third photo | Yes, if they dispute it |
| Earnings and withdrawals | As in 2.5 | As in 2.5 |
| An accident, theft, threat or harassment | **Immediate handover.** Safety first, then facts. | Yes, P1, call them |

### 2.8 Trust and safety

| Situation | Agent does | Owner |
|---|---|---|
| A fake Swap link or site, or someone impersonating Swap | Collects the link or number and a screenshot, and warns the reporter | Yes (takedowns, a public warning) |
| A seller who didn't deliver or is selling prohibited items | Collects details and deal codes | Yes (suspension is yours) |
| "Someone took over my account", "My SIM was swapped" | Treats it as urgent. Advises a password reset if they still can. | Yes, P1. The owner may end sessions. |
| Police or court requests | Never shares data. Takes contact details. | Yes, always |

### 2.9 Privacy and data requests

Under Uganda's Data Protection and Privacy Act, 2019, people can ask to see,
correct or delete their data, or object to its use.

- The agent logs these requests as cases and never answers them with data
  itself.
- Deleting an account is self-service in the apps.
- "Where did you get my number?" can come from sellers the sales agent
  messaged. The answer: the public Jiji listing, on the date recorded in the
  sales agent's database. The agent adds the number to both agents' exclusion
  lists on request.
- Access and correction requests go to the owner. Keep proof of the response
  date. The legal deadline was not verified for this doc; check it.

### 2.10 General questions

How Swap works, fees, the Kampala-only coverage, the UGX 7,000,000 deal cap,
link expiry, and "is Swap licensed?".

On licensing, the agent states only what is true. It must never claim a
licence Swap doesn't hold, and must never confuse Swap with SwApp, which is a
different, Bank of Uganda-licensed company.

## 3. What the agent may do

| Level | What | Examples |
|---|---|---|
| **1. Answer** | Facts from the knowledge base | Fees, how refunds work, how to reset a password |
| **2. Look up** | Read-only checks, once the person is identified (section 4) | Deal status and next step, refund status, withdrawal status, KYC or application status |
| **3. Prepare** | Open a case, gather facts and both parties' statements, and write the owner a brief with a recommended action and a link to the right admin page | Stuck payment, failed refund, a PIN reset request, a dispute summary |
| **4. Never** | Anything that moves money or changes an account | See below |

Level 4, never:

- Moving money in any direction, or promising a refund, release or dispute
  outcome.
- Changing an account: phone, PIN, suspension, KYC decisions, rider approval.
- Asking for or accepting a PIN, password or SMS code. Swap never asks for
  these. The agent says so, and treats anyone who shares one as a warning
  sign.
- Telling a person another person's details: a buyer's number or address, a
  rider's number, a seller's KYC.
- Taking ID documents over WhatsApp.
- Legal advice, or a statement about what a court or regulator would decide.

Changes happen only in the admin console, which already has 2FA, a
permission matrix and an audit trail. The agent's credentials stay read-only,
so if they leak, nobody can move money or take over accounts.

## 4. Identity checks

This is where support at a money platform is most often abused. The common
Ugandan pattern is a SIM swap or a "customer care" impersonator, followed by a
request to reset something.

**Rule: the agent never tells anyone more than they could already see
themselves.**

- **Sellers and riders:** a message from the registered phone number counts
  as the account holder for Level 2 lookups. WhatsApp has already verified
  the number.
  - From any other number, the agent gives only general help, and asks them
    to write from their registered number or use the app.
- **Buyers:** anyone holding the link code can already open the deal page.
  So with the code, the agent may say what the page shows (status and next
  step), and nothing more (no seller phone, no address, no rider details).
  - With the code **and** a WhatsApp number that matches the paying number,
    it may also give the refund destination, masked.
- **High-risk requests** (phone change, PIN reset, unlocking withdrawals,
  "I lost my phone"):
  - The agent never completes these. It collects the request and hands it to
    the owner.
  - The owner verifies before acting. Options: a video call against the KYC
    photo, a fresh Didit check, or the existing SMS code to the new number.
    A request that arrives hours after a SIM change should be treated with
    suspicion.
- **Media:** screenshots of Mobile Money messages are useful evidence. Photos
  of IDs are refused (section 2.4).

## 5. How the agent gets facts

The agent needs read access to deals, payments, refunds, withdrawals, KYC,
rider vetting and disputes, looked up by phone number or link code.

Options:

| Option | For | Against |
|---|---|---|
| A. Log in as an admin with the `support` sub-role | No backend work | Far too much power. `support` has `Disputes: full`, `KYC: full` and `Sellers: full` (migration `0010`), so it can approve KYC, suspend sellers, reset PINs and change phones. It also needs 2FA. |
| B. A read-only Postgres role on the production database | Quick | Couples the agent to table layouts, and bypasses the API's authorization and audit. A leaked credential exposes whole tables. |
| **C. A small support API in the backend (recommended)** | Least privilege. Every lookup is audited. Returns only the fields the agent needs. Fits the repo's own rules. | Needs a few endpoints and a machine credential |

Recommended: **C**, with a new `/v1/support/*` router. Authentication is a
long random API key, stored hashed, optionally limited to the VPS IP, and
separate from staff logins. Endpoints:

- `GET /v1/support/lookup?phone=` returns:
  - whether the number is a seller, a rider or a buyer on recent deals;
  - a profile summary (status, KYC, suspended or not, lock state);
  - open deals, with code, state, amount, item label and age.
- `GET /v1/support/deals/{link_code}`: state, timeline, payment, refund and
  dispute status, with no personal data of the other party.
- `GET /v1/support/withdrawals?profile=`: recent withdrawals and their status.
- `GET /v1/support/riders/{phone}/application`: vetting state.
- `GET /v1/support/watch`: the proactive queue (section 7).
- `POST /v1/support/cases`, for when cases are mirrored into Swap for the
  admin console (phase 2).

Each call writes an audit event (`actor_type = support_agent`, with the case
number). Phone numbers are masked in responses except where the agent needs
them to match the sender.

## 6. Cases, priorities and response times

A **case** is one problem for one person: say "my refund". It can span
several messages and days.

- **What a case records:** number, person, category (section 2), priority,
  status, linked deals, summary, what's been done, the owner's decision, and
  timestamps.
- **Statuses:** `open`, `waiting_on_customer`, `waiting_on_owner`, `resolved`.
  A resolved case reopens if the person writes again within 7 days about the
  same thing.
- **A complaints register:** the case table doubles as one. Payment-sector
  consumer protection rules in Uganda (the National Payment Systems (Consumer
  Protection) Regulations, 2022) set complaint-handling duties for licensed
  payment businesses.
  - Check with a lawyer whether they apply to Swap and what deadlines they
    set. This doc does not quote their day counts because they could not be
    verified here.
  - Keeping a dated record of every complaint and its outcome is wise either
    way.

Priorities and targets. These are internal targets, not legal ones.

| Priority | Examples | Agent's first reply | Owner |
|---|---|---|---|
| **P1** money or safety | A payment or withdrawal stuck over the threshold, a failed refund, account takeover, rider safety | Within minutes, 24/7 | Alert at once, day or night |
| **P2** blocked | PIN lock, KYC stuck, phone change, a dispute opened, a funded deal with no rider | Within minutes during hours | Within the day |
| **P3** questions | How-to questions, fees | Within minutes during hours | Never, unless the agent can't answer |

The owner gets a **brief**, not a chat log:

- who;
- what they want;
- what the agent checked (facts, with deal codes);
- what the agent has told them;
- the recommended action, with a link to the right admin page;
- the message the agent will send once it's done.

After the owner acts, the owner runs `./agent done <case>`, or the agent sees
the state change in a lookup. The agent then tells the person. After 24 hours
of silence that needs a WhatsApp template (section 8).

A morning and evening digest lists open cases, what's waiting on the owner,
and anything close to its target.

## 7. Proactive support

The cheapest support is telling people before they ask. Every few minutes the
agent reads `/v1/support/watch` and acts on:

| Signal | Action |
|---|---|
| A payment `pending` or `indeterminate` for over 15 minutes | Tells the buyer it's being checked; opens a P1 case after 30 minutes |
| A deal `FUNDED` with no rider for over 30 minutes | Alerts the owner (G1) and tells the buyer and seller |
| A refund failure (`refund_failed_alert`) | P1 case, and tells the buyer the refund is being fixed |
| A withdrawal `sent` for over 2 hours | Tells the owner |
| A dispute opened | Tells both parties what to submit; case brief to the owner |
| A seller's KYC waiting for manual review over 24 hours | Reminds the owner |
| A rider application waiting | Reminds the owner |
| SMS credit low (`credit_balance_probe`) | Alerts the owner. Sign-ups and resets stop without SMS. |

Proactive messages to someone who hasn't written in the last 24 hours need an
approved WhatsApp utility template.

## 8. Channels and hours

**WhatsApp is the main channel.** It should use a separate, official Swap
Support number: not the sales agent's number and not the owner's. Put it in
the apps next to the email link: `wa.me/<number>` on the seller account page,
the PIN and withdrawal lock screens, the rider app and the buyer deal page.

**Use the official WhatsApp Business Platform (Cloud API) for support, not
whatsapp-web.js.**

- **Allowed use:** Meta's AI policy (in force for all businesses since 15
  January 2026) bans general-purpose AI chatbots on the Business Platform. It
  explicitly allows businesses to use AI for their own customer support.
- **The number is too valuable to risk:** the support number will be printed
  in the apps. Losing it to a ban costs far more than a sales number.
- **Cost:** on the per-message pricing Meta uses since July 2025, replies
  within 24 hours of the customer's message were free until 30 September
  2026. **From 1 October 2026 each business number gets 1,000 free service
  messages a month.** After that, the country's utility rate applies, and
  in-window utility replies are no longer free. At Swap's size, 1,000 a month
  should cover early support. Check the Uganda utility rate before relying on
  it.
- **Trust:** a verified business profile tells buyers they're talking to the
  real Swap, which matters in a market full of impersonators.
- **Needs:**
  - Meta business verification;
  - a public HTTPS webhook (the VPS already runs Caddy);
  - utility templates for follow-ups after 24 hours (case updates, "your
    refund was sent").

The messaging port from the sales agent carries over. The Cloud API becomes a
new adapter alongside the bridge. The sales agent planned this move after
funding anyway.

**Email** (`support@swapafrica.online`, Zoho) and the **contact form** should
feed the same case list later:

- IMAP polling for email;
- for the form, change `/api/contact` to post to the agent instead of (or as
  well as) Pushover.

**Be open that it's an assistant.** A support agent should say it is Swap's
assistant and that a person can take over. Sellers trust a named, official
support line more than a person who might be a scammer.

**Hours.** Support is not the sales agent's human routine. The agent answers
promptly whenever the AI is available.

- Suggested hours: full answers 07:00 to 22:00.
- Outside those hours: an instant acknowledgement with the expected reply
  time.
- P1 alerts reach the owner at any hour.
- The 10-minute cron from the sales agent is too slow for support. Use the
  webhook for inbound messages, plus a short loop.

## 9. The AI model and customer data

Support messages contain personal data: names, numbers, amounts, addresses,
and sometimes screenshots.

- **The model's terms.** Google's Antigravity terms say prompts from personal
  plans may be used to improve their models. Accounts accessed through Google
  Workspace or Google Cloud are excluded. Whether a settings toggle prevents
  this is unclear in public forum threads (see sources). Sending customer
  data to `agy` on a personal Google AI Pro plan is therefore a data
  protection risk.
- **Processing outside Uganda.** The Data Protection and Privacy Act allows
  processing outside Uganda only with adequate protection or the person's
  consent (section 19).

What to do:

- **Keep personal data out of prompts.** Phone numbers never go in, as in the
  sales agent. Names become "the buyer" and "the seller". Deal facts go in
  as a short summary from the lookup. Addresses stay out unless needed.
- **Never send ID documents to the model.** Payment screenshots only when
  needed.
- **Update the privacy policy** before launch. It should say that support
  messages are handled with an AI assistant and that the provider may process
  them outside Uganda.
- **Move support to the paid Gemini API early.** The paid API tier is not used
  for training. Support is a smaller, higher-value volume than sales. The
  sales agent's `LLM_PROVIDER=gemini` adapter is already built, including
  voice notes and pictures.
- **Keep the AI gate.** Budget, cool-down and the ability to rest the AI all
  carry over. While the AI rests, the agent still acknowledges messages and
  handles P1 alerts without it.

## 10. What to reuse from the sales agent

| Sales agent part | In the support agent |
|---|---|
| Ports: `llm/`, `messaging/`, SQLite store, `jobs/common.py`, AI gate, `ask_json` | Keep as they are |
| The `agy` and Gemini adapters, with attachments | Keep. Voice notes and screenshots are common in support. |
| WhatsApp bridge (hidden numbers, media) | Keep for development. Add a Cloud API messenger for production. |
| `agent/guard.py` | Keep the approach and change the rules: allow the Swap links and the support email; forbid promises ("you will get a refund", "guaranteed") and requests for PIN or code; add masking checks for numbers |
| Contacts, messages, alerts, the one-reply-per-inbound-message index, claim-before-send | Keep. Add `cases`, `case_events` and `lookups` tables. |
| Longest-waiting first, the per-check time limit, owner takeover, `release` | Keep |
| `tick`, human-hours sessions, presence and typing, first messages, Jiji, drafting | Drop. Support answers inbound messages promptly and never sends cold messages. |
| `knowledge/swap.md` | Grows into a support knowledge base: one file per area in section 2, with the exact rules from the code (limits, fees, timings) |
| New | A support API client, case manager, identity check (section 4), proactive watcher (section 7), owner briefs and digest, templates for follow-ups after 24 hours |

**Shared code.** Copy the sales agent's `swap_sales` core into
`swap_support` rather than sharing a package, for now. The two will drift
(schedules, guard rules), and a shared library can come later once both are
stable.

## 11. Gaps in the Swap platform that support will hit

The support agent can explain things, but it can't fix gaps in the platform.
These are worth fixing before or alongside the agent.

| # | Gap | Effect on support | Fix |
|---|---|---|---|
| **G1** | There's no way to cancel a funded goods deal before pickup. The state machine allows `FUNDED` / `RIDER_ASSIGNED` / `AT_PICKUP` to `VOIDED`, but the only void paths in code are photo rejection, refusal at the door and the unpaid-link sweep. | Money is stuck when no rider comes, the item is sold out, or the buyer cancels before pickup. Today the only fix is editing the database by hand. | An admin "void and refund" action (`Refunds: edit`, reason required, audited), and maybe a seller "cancel, item unavailable" action |
| **G2** | A failed buyer refund only writes an audit event (`refund_failed_alert`, owner-only log) | Nobody is told. The money sits in `refund_liability`. | A failed-refunds queue in the admin app, retry to the same or a corrected number (verified), and a message to the buyer |
| **G3** | The admin escrow list filters only by status. There's no search by phone or link code. | Every support question starts with "which deal?" | Search by link code and phone. The support API (section 5) covers this for the agent. |
| **G4** | Buyers get no SMS or WhatsApp updates. They see progress only on the link page. | Most "where is my order?" questions come from this | Short transactional SMS or WhatsApp templates at: payment confirmed, photos ready, rider on the way, refund sent |
| **G5** | Support contact is email and Pushover only, with no WhatsApp number in the apps | People end up on the owner's personal WhatsApp | Add the support WhatsApp link (section 8) |
| **G6** | The terms and the code disagree. The terms say "either party may open a dispute" (only buyers can), that orders can be "canceled before dispatch" (no path, see G1), and that the delivery fee is kept on refusal "if the seller fulfilled the description accurately" (the code always keeps it). The terms in `docs/legal` also link the privacy policy to a file on a local Windows disk. | The agent would have to either contradict the terms or explain behaviour the terms don't describe | Align the terms and the code |
| **G7** | Rider provisioning, when `RIDER_ACTIVATION_BY_CODE` is off, texts a temporary password `SwapAfrica@<3 digits>`: only 1,000 possibilities | A security risk. Support will also get "I didn't get my password". | Turn on activation by code (Phase 1d) and stop sending passwords by SMS |
| **G8** | The `support` staff sub-role can't resolve disputes (it needs `Refunds: edit`) but can approve KYC and reset PINs | Fine while the owner is alone. It matters when you hire a support person. | Review the matrix before inviting support staff |
| **G9** | The privacy policy doesn't mention AI-assisted support or processing outside Uganda | Data protection exposure (section 9) | Update the policy |
| **G10** | Check whether the payment consumer protection rules apply to Swap | Complaint deadlines and records | Legal check (section 6) |

## 12. Decisions for the owner

1. **Channel:** start on the official WhatsApp Cloud API (recommended), or on
   the bridge first and move later?
2. **Model:** is `agy` on the personal plan acceptable for customer data, or
   move support to the paid Gemini API from day one (recommended)?
3. **Support API (section 5):** build it in the Swap backend (recommended),
   or use a read-only database role to start?
4. **Hours and P1 alerts:** are 07:00 to 22:00 and "P1 at any hour" right?
5. **Identity for high-risk requests:** which check before a PIN reset or
   phone change: a video call against KYC, a fresh Didit check, or both?
6. **Gaps:** which of G1 to G10 to fix first. G1 and G2 are about money
   stuck with no way out.
7. **Disclosure:** introduce the agent as "Swap Support assistant" and offer
   a person on request (recommended)?

## 13. Build plan

1. **Swap backend** (in the `swap` repo, following its AGENTS.md):
   - G1: admin void and refund;
   - G2: failed refunds queue and retry;
   - the read-only `/v1/support/*` API with an audited machine key.
2. **Agent core:** copy the sales agent's ports, store and AI gate, then add:
   - cases, the identity check, the support API client;
   - the knowledge base by area;
   - the guard rules for support.

   Tests with fake lookups for every row in section 2.
3. **Dry run:** replay realistic conversations: voice notes, screenshots,
   Luganda and English, fake links, PIN requests, SIM-swap stories. Tune the
   prompts and knowledge base.
4. **Channel:** Cloud API adapter, webhook on the VPS, utility templates,
   support number in the apps.
5. **Proactive watcher and digest** (section 7).
6. **Later:** email and contact-form intake; cases mirrored into the admin
   console; a human support hire using the same cases.

## Sources

- The Swap code and docs in `aaron-official/swap`, as cited in each section.
- **WhatsApp AI policy:** [respond.io: WhatsApp AI chatbot policy](https://respond.io/blog/whatsapp-ai-chatbot-policy),
  [MediaNama: WhatsApp bans external AI providers from the Business API](https://www.medianama.com/2025/10/223-whatsapp-bans-external-ai-providers-business-api/).
- **WhatsApp pricing:** [Meta: Pricing on the WhatsApp Business Platform](https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing),
  [EngageLab: WhatsApp Business API pricing 2026 and the 1 October changes](https://www.engagelab.com/blog/whatsapp-business-api-pricing).
- **Uganda payment consumer protection:** [National Payment Systems (Consumer Protection) Regulations, 2022 (ULII)](https://ulii.org/akn/ug/act/si/2022/103/eng@2022-09-09).
  Not read in full here; its deadlines still need checking.
- **Uganda data protection:** [Data Protection and Privacy Act, 2019 (ULII)](https://www.ulii.org/akn/ug/act/2019/9),
  [DLA Piper summary](https://www.dlapiperdataprotection.com/index.html?t=law&c=UG).
- **Mobile Money reversals:** [Pulse Uganda: reversing money sent to the wrong number](https://www.pulse.ug/story/how-to-reverse-airtel-money-sent-to-wrong-account-on-airtel-mtn-2024120410112087221).
- **Antigravity data use:** [Google AI forum: Antigravity data training opt-out](https://discuss.ai.google.dev/t/antigravity-data-training-opt-out/125236),
  [Google AI forum: data privacy for commercial use](https://discuss.ai.google.dev/t/data-privacy-for-commercial-use-how-to-access-antigravity-with-full-protections/132828).
