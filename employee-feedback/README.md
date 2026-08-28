# Employee engagement pulse

A weekly wellbeing pulse that actually closes the loop. An employee taps a face and
writes a line; the response is matched to their employee record, rated for severity by
an AI node, and mailed to their HR business partner with an **Mark as attended** button.
If nobody clicks that button within two days, the business partner gets one reminder.

Two pieces: a self-contained HTML form (`index.html`, no build step) and one n8n
workflow that does dispatch, intake, the attended callback and the reminder sweep.

## The form

**Page 1** — four animated faces: happy, excited, sad, angry. One tap, and the choice
registers with a mood score (angry 1 … excited 5) so the numbers trend over weeks even
when nobody writes anything.

**Page 2 changes with the answer.** Pick *sad* and it asks "Sorry it has been heavy. Do
you mind telling us more?" with chips for Workload, My manager, Career direction, Rather
not say. Pick *excited* and it asks what the good news is, with chips for wins,
recognition and growth. Same for the placeholder text and the thank-you screen — the
angry path says the case escalates if conduct or safety is involved, the sad path
acknowledges that writing it took something. All of it is in the `COPY` object at the
top of the script; edit it there.

Then the reach-out question (yes / maybe / no) and a work email, which is what the
workflow matches on.

### The faces are SVG, not GIFs

They animate — happy bobs and blinks, excited bounces with twinkling sparkles, sad
droops with a tear that falls on a loop, angry vibrates with steam puffs — but they are
inline SVG rather than image files. No external requests, nothing to break behind a
strict CSP, ~2KB of markup, and `prefers-reduced-motion` turns the motion off.

To use real GIFs instead, put a URL or data URI in the `gif` field of any mood in the
`MOODS` object. The form then uses your image everywhere the SVG appeared:

```js
const MOODS = {
  sad: { label:"Sad", score:2, accent:"var(--m-sad)", gif:"https://…/sad.gif" },
  …
};
```

## Wiring it up

1. Create two n8n data tables:

   **`employees`** — `employee_id`, `full_name`, `email`, `unit`, `job_title`,
   `line_manager_name`, `line_manager_email`, `bp_name`, `bp_email` (all string),
   `is_active`, `receives_pulse` (boolean), `date_joined` (date).

   **`pulse_cases`** — `case_id`, `employee_email`, `employee_name`, `unit`,
   `line_manager_email`, `bp_email`, `bp_name`, `feeling`, `drivers`, `share_text`,
   `severity`, `severity_reason`, `status`, `attended_note`, `token` (string),
   `submitted_at`, `attended_at` (date), `reminder_sent`, `escalated` (boolean).

2. Import `n8n/employee-engagement-pulse.json`.

3. Reconnect what the export strips: a Gmail credential on the six Gmail nodes, an
   OpenAI credential on **Severity Model**, and the two data tables on every Data table
   node (the picker is empty — `employees` on two nodes, `pulse_cases` on five).

4. Replace `REPLACE-WITH-HR-LEAD@example.com` in **Escalate to HR Lead** and **Alert
   Unmatched Submission**.

5. Publish the workflow, then fix the three URLs:
   - copy the **pulse-submit** production URL into `WEBHOOK_URL` at the top of the
     `<script>` block in `index.html`;
   - put your n8n host into the `attended_url` field of **Assemble Case** and into the
     button URL in **Send Reminder to Business Partner**;
   - put wherever you host `index.html` into the link in **Send Pulse Invite**.

Leave `WEBHOOK_URL` empty and the form still runs end to end — it just tells the person
nothing was sent, and logs the payload to the console.

## What gets sent

```json
{
  "submittedAt": "2026-08-28T09:12:04.001Z",
  "secondsTaken": 38,
  "employee_id": "EMP-001",
  "work_email": "someone@company.com",
  "mood": "sad",
  "mood_label": "Sad",
  "mood_score": 2,
  "drivers": ["Workload", "Deadlines"],
  "details": "Been carrying the new project alone since the handover.",
  "wants_outreach": "yes"
}
```

`mood` is the machine key and `mood_label` the display name, so renaming a face in the
UI never breaks stored data. `drivers` is an array; the workflow joins it into a string
for the case row and the email.

## How the workflow is laid out

One canvas, four sticky-note sections, four triggers:

| Section | Trigger | What it does |
| --- | --- | --- |
| 1. Friday dispatch | Schedule, Fri 09:00 | Mails the form link to every active employee flagged `receives_pulse`, with `employee_id` and `email` in the query string |
| 2. Submission handler | Webhook `POST /pulse-submit` | Looks up the employee, rates severity, logs the case, mails the business partner |
| 3. Attended callback | Webhook `GET /pulse-attended` | Validates `case_id` + `token`, marks the case attended, returns a confirmation page |
| 4. Reminder sweep | Schedule, daily 09:00 | One reminder for anything pending past two days, then flags `reminder_sent` |

### Design decisions worth keeping

**Routing is data, not an IF.** The business partner comes from `bp_email` on the
employee row. A BP hire, exit or reassignment is a data edit, never a workflow edit.
The only severity branch is `critical`, which additionally pings the HR lead.

**The lookup matches email OR employee_id**, so a submission still lands if someone
types a personal address or the link loses its query string. n8n data table filters are
all-AND or all-OR and cannot mix, so the `is_active` check lives in the **Employee
Record Found?** IF rather than the query. Inactive or unmatched submissions go to an
alert mail instead of being silently dropped.

**State lives in `pulse_cases`, not in a waiting execution.** A `Wait` node with a
two-day resume would be fewer nodes, but the execution sits open for 48 hours and an n8n
restart orphans the case. A status row survives restarts and can be reported on.

**The AI sees mood, score, drivers and comment together**, which matters most when
someone taps *angry* and writes nothing at all. It returns `severity`, a one-sentence
`reason`, a `theme`, and `needs_response`, and is instructed not to diagnose or
speculate about health.

## Notes

- The form is not anonymous and says so on both pages. A pulse that routes to a named
  business partner and gets AI-scored cannot honestly claim anonymity.
- The attended link is authorised by a `case_id` + `token` pair generated per
  submission. Guess-resistant, but not signed — worth hardening if BP mail leaves your
  domain.
- The submit webhook is unauthenticated so any host can POST to it. Fine for an
  internal pilot; add header auth before a company-wide rollout.
- `allowedOrigins: "*"` on the webhook is what lets a page on another host POST at all.
  Without it the browser preflight fails and every submission is silently lost.
- Activating the workflow arms all four triggers at once, and every execution lands in
  one list. That is the trade for having the whole process on one canvas.
