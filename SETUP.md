# KnitLab GRN — Draft GRN workflow, notifications & WhatsApp

The app is now **online-only**. There is no IndexedDB/Dexie, no offline queue and no `TEMP-` numbers.
Every read and write goes straight to Supabase (database + Storage), and Supabase Realtime keeps every open screen live.

## Workflow

1. **Step 1 – Quick intake** (gate / receiving point, `GRNs → Quick intake`)
   Supplier, material, photo (camera on phones), gate pass no, DO no, received date/time → saved as `status = 'DRAFT'`.
   The photo goes to the private `grn-photos` Storage bucket. No GRN number and no stock yet.
2. **Step 2 – Inspection** (stores / quality, `GRNs → Pending drafts`)
   Opens the full form with the photo and gate details. Quantities, weights, quality status and remarks are entered,
   then **Complete GRN** sets `status = 'COMPLETED'`. The database trigger issues `GRN-YYYY-00001` and the
   items now count towards RM Store stock (stock = accepted qty of COMPLETED, in-store GRNs − issues − returns).
3. Admins can void (`VOIDED`) / restore / delete. Staff can delete their own drafts.

## Deploy checklist

1. **Database** – run, in order, in the Supabase SQL editor (or via Lovable Cloud):
   `drizzle/migrations/0006_roles_and_audit_log.sql` then `0007_draft_grn_workflow.sql`.
   Existing GRNs are migrated: `active → COMPLETED`, `voided → VOIDED`.
2. **Dependencies** – `bun install` (or `npm i`). `dexie` was removed from `package.json`.
3. **Twilio** (WhatsApp)
   - Quick test: Twilio Console → Messaging → *Try it out → Send a WhatsApp message* (sandbox, `+14155238886`).
     Every recipient must first send the sandbox join code from their WhatsApp.
   - Production: an approved WhatsApp sender. Messages that start a conversation outside the 24-hour window must use an
     **approved template**; create one with variables `{{1}}` supplier, `{{2}}` time, `{{3}}` photo (Yes/No) and set `TWILIO_CONTENT_SID`.
4. **Edge Function secrets** (`supabase secrets set …`, or Lovable Cloud → Secrets):

   | Secret | Value |
   |---|---|
   | `WEBHOOK_SECRET` | long random string (you invent it) |
   | `TWILIO_ACCOUNT_SID` | `AC…` |
   | `TWILIO_AUTH_TOKEN` | Twilio auth token |
   | `TWILIO_WHATSAPP_FROM` | `+14155238886` (sandbox) or your sender |
   | `TWILIO_CONTENT_SID` | *(optional)* approved template SID `HX…` |
   | `APP_TIMEZONE` | *(optional)* default `Asia/Colombo` |

   `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are provided automatically.
5. **Deploy functions** – `supabase functions deploy notify-draft-grn` and `supabase functions deploy twilio-status`
   (`supabase/config.toml` already sets `verify_jwt = false`; both functions check their own secret / Twilio signature).
6. **Webhook** – edit and run `supabase/webhooks/draft_grn_whatsapp.sql` (needs `<PROJECT_REF>` and the same `WEBHOOK_SECRET`).
7. **Users** – sign in as admin → `Admin → Users`: enter each person's number in E.164 (`+94771234567`), then switch
   *WhatsApp alerts* and/or *In-app alerts* on. Results appear under `Admin → WhatsApp`.

## Test the WhatsApp function

```sh
curl -X POST "https://<PROJECT_REF>.supabase.co/functions/v1/notify-draft-grn" \
  -H "Content-Type: application/json" -H "x-webhook-secret: <WEBHOOK_SECRET>" \
  -d '{"type":"INSERT","record":{"id":"00000000-0000-0000-0000-000000000001","status":"DRAFT","grn_date":"2026-10-08","received_time":"15:31","photo_url":"x/y.jpg"}}'
```

## Notes

- In-app: a new draft by someone else raises a toast (users with *In-app alerts* on) and updates the header bell,
  which shows the number of drafts waiting for inspection.
- The legacy `grns.temp_ref` column is kept only so old paper slips can still be matched; the app no longer reads or writes it.
- The WhatsApp alert is sent once per draft (a retry of the webhook is ignored). Twilio status callbacks move the log row
  `queued → sent → delivered → read` or `failed / undelivered`.
