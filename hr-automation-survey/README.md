# HR automation survey

A two-question survey that asks which HR processes people would hand to AI first,
and what makes those processes painful. One self-contained HTML file, no build step,
no dependencies — open `index.html` and it runs.

Responses post to an n8n webhook, which appends one row per response to Google Sheets.

## The questions

1. **Which TWO HR processes would you most likely automate or improve using AI?**
   Fourteen options plus Other, capped at two selections — pick a third and it is
   refused, so the data stays a genuine ranking instead of a wish list.
2. **What makes these processes a good candidate for AI or automation?**
   Nine options plus Other, select as many as apply. The two picks from Q1 are shown
   as chips in the header so "these processes" always has a referent.

Knowing that Recruitment was picked 40 times is much less useful than knowing it was
picked because approvals pile up and nobody can track follow-ups. Q2 is where the
build decisions actually come from.

## Wiring it up

1. Import `n8n/hr-survey-to-google-sheets.json` into n8n.
2. Connect a Google Sheets credential on **Append Response Row**, then pick the
   spreadsheet and tab (both were stripped from the export).
3. Put this header row in row 1 of that tab, spelled exactly like this — the append
   node matches on column names, so a typo sends data to the wrong column:

   ```
   Submitted At	Seconds Taken	Priority 1	Priority 2	All Picks	Other Process	Reasons	Other Reason	Reason Count	Referrer
   ```

4. Publish the workflow and copy the **Production** webhook URL.
5. Paste it into `WEBHOOK_URL` at the top of the `<script>` block in `index.html`.

Leave `WEBHOOK_URL` empty and the survey still runs end to end — it just tells the
respondent their answers stayed on the device instead of sending them anywhere.

## What gets sent

```json
{
  "submittedAt": "2026-08-25T10:12:04.001Z",
  "secondsTaken": 46,
  "processes": ["Onboarding & Offboarding", "Other"],
  "processOther": "Exit interviews",
  "processesResolved": ["Onboarding & Offboarding", "Exit interviews"],
  "reasons": ["Too much manual/repetitive work"],
  "reasonOther": null,
  "reasonsResolved": ["Too much manual/repetitive work"],
  "referrer": null
}
```

`processes` and `reasons` keep the raw option labels, which is what you want for
counting. The `*Resolved` arrays substitute the free-text answer wherever someone
picked Other, which is what you want for reading. The workflow splits
`processesResolved` into **Priority 1** and **Priority 2** columns so first choices
can be counted separately from second choices.

## Notes

- No email, no name, no sign-in. The survey collects nothing that identifies a
  respondent, and the start screen says so.
- Free text is HTML-escaped before it reaches the DOM.
- Honours `prefers-reduced-motion` — the animations and confetti turn themselves off.
- The webhook allows all origins (`allowedOrigins: "*"`), which is what lets a page on
  any host POST to it. Without that setting the browser preflight fails and every
  submission is silently lost.
