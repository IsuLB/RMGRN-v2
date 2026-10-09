<!-- LOVABLE:BEGIN -->
> [!IMPORTANT]
> This project is connected to [Lovable](https://lovable.dev). Avoid rewriting
> published git history — force pushing, or rebasing/amending/squashing commits
> that are already pushed — as it rewrites history on Lovable's side and the
> user will likely lose their project history.
>
> Commits you push to the connected branch sync back to Lovable and show up in
> the editor, so keep the branch in a working state.
<!-- LOVABLE:END -->

## Architecture rules
- The app is online-only: all data goes straight to Supabase (`src/lib/data.ts` reads with TanStack Query, `src/lib/grn-actions.ts` writes). No Dexie / IndexedDB / offline queue / TEMP numbers.
- `src/hooks/use-realtime.tsx` subscribes to Supabase Realtime, refreshes cached lists and raises the "new draft GRN" toast; `NotificationBell` shows pending drafts.
- GRN lifecycle: `DRAFT` (quick intake + photo in the private `grn-photos` bucket) -> `COMPLETED` (inspection done; the `assign_grn_no` trigger issues GRN-YYYY-00001 at this moment) -> `VOIDED` (admin only, restorable). Staff can delete their own drafts; admins can delete any GRN.
- Inventory is derived on the fly from COMPLETED, in-store GRN items (accepted qty) minus issues and returns; drafts, voided GRNs and direct handovers never count. The DB also rejects over-issue / over-return.
- Roles: `profiles.role` (admin/staff). Only admins change roles, notification settings, void/restore GRNs, read the audit log and WhatsApp log. Enforced in RLS and triggers; the UI only hides buttons.
- WhatsApp: a Database Webhook on new DRAFT GRNs calls the `notify-draft-grn` Edge Function (Twilio); `twilio-status` records delivery status in `whatsapp_logs`. See SETUP.md.
- `audit_log` is written only by database triggers (SECURITY DEFINER).
- Company print header (name, subtitle, logo data URL) is stored per device in localStorage (`src/lib/company.ts`).
- RLS on shared tables uses `public.is_staff(auth.uid())` (user has a profile) — all staff share data.
