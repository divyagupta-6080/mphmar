# mustbeus — Status, Fixed Issues, Pending Actions & Expected Behaviour

_Generated: 2026-08-24. Deploy in progress (all 10 backend services rebuilt + redeployed)._

This is the single consolidated document covering:
1. What's fixed (and deployed)
2. What still needs fixing (with owner/action)
3. Expected behaviour after each fix
4. Ops/infra action items that only your team can do (not code)

Production: `http://mustbeus-1075426823.us-east-1.elb.amazonaws.com`

---

## 1. FIXED & DEPLOYED

| Area | Issue | Expected behaviour now |
|---|---|---|
| Auth | Org registration showed no confirmation | Org signup → "verify email" page + success popup, same as volunteer |
| Auth | Logout bounced login→landing | Clean logout to landing, no flicker |
| Auth | Login didn't flag unverified email | Unverified login → clear "verify your email" message |
| Auth | Re-registering unverified email dead-ended | Re-sends verification link instead of erroring |
| Auth | Forgot/reset password errors unclear | Clear error detail (email delivery still blocked — see ops) |
| Auth | City field not usable | City typeahead autocomplete works |
| Org profile | Edits not saved / not reflected | Profile edits (incl. year founded) save + show immediately |
| Attendance | Check-in / check-out 500 | Check-in/out succeeds |
| Opportunities | Pics not showing / could post w/o pics | Pics render; publishing requires ≥1 photo |
| Opportunities | Skill filter broken | Multi-select skill filter works (OR match) |
| Opportunities | Date off-by-one | Correct dates on detail/apply |
| Messaging | Deep link + Message button inert | Message button opens the right conversation |
| Referrals | Refer volunteer/org did nothing | Referral submits + best-effort email (delivery blocked — see ops) |
| Payments | Yearly discount not shown | Real yearly saving % shown |
| Payments | Professional plan hard 500 | Graceful error (completion still needs Stripe IDs — see ops) |
| Account | Org treated as volunteer (tossdivya) | Role corrected in Keycloak; org sees Post Opportunity after re-login |

## 2. FIXED IN CODE — GOING LIVE WITH THIS DEPLOY

These were coded + compiled this session and are in the deploy that's running now.

| Area | Fix | Expected behaviour after deploy |
|---|---|---|
| Opportunities | Published-edit restriction relaxed to only city/country | Orgs can edit a published opportunity's title/skills/cause areas/type; only location locked |
| Gamification | Engagement badge criteria (first/10/25/50/100 opportunities) + scoring `total_projects` | Posting opportunities awards org XP, badges, recent-activity |
| Calendar | DatePickerField selection fix | Volunteers can pick start date; same-day selectable |
| Payments | Two-plan chooser at `/checkout`; all upgrade buttons point there | Org "Upgrade" shows BOTH Superhero + Professional to choose |
| Auth | Role-aware duplicate-registration messages | Registering an email that exists as the other role → clear "log in as X / use different email" message. Volunteer & org stay permanently separate (1 email = 1 role) |
| Notifications | Org now notified on badge earned; org auto-provisioned into `users.users` | Orgs receive notifications (badges now; foundation for all org notifications) |
| Messaging | `subscription_tier` JWT claim mapper + declared attribute (Keycloak, already live) | Paid orgs/volunteers can START conversations (was blocked for everyone) |
| Messaging | messenger self-provisions Vault transit key on startup | Works once Vault policy allows key creation (see ops action A1) |

## 3. STILL NEEDS FIXING (code) — PENDING ACTION ITEMS

| # | Issue | Action | Expected behaviour when done |
|---|---|---|---|
| P1 | Support ticket "Unexpected token '<'" | Build `/api/support/ticket` backend (mirror referrals) + point `Support.tsx` at API client | Support form submits, shows success |
| P2 | WebSocket `/wss/messenger` 403 (real-time chat) | Investigate WS auth in `messenger-service/app/routers/ws.py` (token in query string rejected) | Messages appear live without refresh |
| P3 | Org notifications beyond badges (application events, check-in) | Extend the org `users.users` auto-provision to the `application.status_changed` org path too | Orgs get ALL notification types |
| P4 | Hero-story admin alert | Verify `content.story.submitted` → admin notification delivery | Admins pinged on new pending story (queue already works) |
| P5 | "Upcoming opportunity not working" (test doc #12) | Reproduce + diagnose | Upcoming opportunities list/section works |
| P6 | Phantom "already applied" on new org (test doc #20) | Reproduce + diagnose | No false "applied" state |
| P7 | Top-corner arrows look/clickability (test doc #9) | UI fix | Arrows clearly clickable |

## 4. OPS / INFRA ACTIONS (NOT CODE — only your team can do these)

| # | Action | Why | Effect |
|---|---|---|---|
| A1 | **Grant the messenger Vault AppRole `create` on `transit/keys/messenger-content`** (or run `vault write -f transit/keys/messenger-content type=aes256-gcm96 auto_rotate_period=720h` once). Vault IS running — this is a POLICY grant, not an outage. | messenger encrypts message bodies with this key; it was never created, and the AppRole can only `read`, so the service can't self-create it. | Message SEND works immediately (currently 500). This is THE blocker for sending. |
| A2 | Set Stripe price IDs in Vault (`org_professional_monthly/yearly_price_id`) | Professional checkout can't create a Stripe session without them | Professional plan checkout completes |
| A3 | Restore SMTP/SES | Email sending is down | Verification, reset, referral, support emails actually deliver |
| A4 | Assign `admin` realm role to a chosen account | Admin portal `/admin` is gated to `admin`; nobody has it → everyone bounced to landing | That account can access the admin portal + story moderation |

## 5. WORKING AS DESIGNED (not bugs — clarifications)

| Topic | Clarification |
|---|---|
| Hero story "where does it go for review" | Saved as `pending`, appears in admin portal `/admin/stories` (filter Pending) + moderation-queue count. Not lost. |
| "Messaging not working for anyone" | Correct behaviour: only PAID tiers can START a conversation (free = 402). After the Keycloak fixes, paid users can start chats; sending needs A1. |
| Admin portal not visible | `/admin` requires the `admin` role (A4). Not a code bug. |
| tossdivya org role | Account was on `volunteer`; corrected to `organization`. Registration code is sound — likely registered via the default Volunteer tab. |

## 6. MESSAGING — precise chain status (most-asked area)

Messaging was blocked by FOUR independent issues, found by live end-to-end testing:

1. ✅ No `subscription_tier` JWT claim mapper → everyone treated as free → 402 on start. **Fixed (Keycloak).**
2. ✅ `subscription_tier` not a writable user-profile attribute → couldn't set paid tier. **Fixed (Keycloak).**
3. 🚫 Vault transit key `messenger-content` never created → SEND 500. Code self-provisions on startup but the AppRole lacks `create` permission → **needs ops action A1.**
4. 🔎 WebSocket 403 → real-time delivery. **Pending P2.**

After #1+#2 (live): a paid org can CREATE a conversation (verified — it shows in the inbox). SEND is blocked only by #3 (A1). Do A1 and messaging works end-to-end for paid users.

## 7. TEST ACCOUNT STATE (for your testing)

- Org `tossdivya@gmail.com` — role `organization`, currently `subscription_tier=professional` (set for messaging test; tell me to revert to `free` if it shouldn't be paid). Password as you provided.
- Volunteer `divya21071998@gmail.com` — role `volunteer`, free. Password as you provided.
- Requested deletions of both accounts for a clean end-to-end re-test are **NOT done** (awaiting your explicit go — irreversible).

## 8. Rollback / backup references

- Keycloak config backup (client scopes + user profile BEFORE mapper/attribute changes): `_deploy-backups/keycloak-mapper-20260929-201800/`
- Pre-deploy image digests: `_deploy-backups/predeploy-20260929-204532/ecr-digests-before.txt`
- Deploy log: `_deploy-backups/deploy-log-20260929.txt`
