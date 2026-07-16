# Respec Platform ↔ n8n Integration

How customer notifications leave the platform and reach n8n.

- **Supabase project:** `respec-platform` (`cnpyltipddxfjyucigxj`)
- **Endpoint the DB calls:** stored in `platform_settings.n8n_notify_url`
  (currently `https://n8n.braxtonjeht.com/webhook/platform-notify`)

## Why this differs from Josh's dashboard

`josh-ecu-dashboard` POSTs to `https://n8n.braxtonjeht.com/webhook/ecu-status-change`
**from the browser**, with the URL sitting in client-side JS. That's workable for a
single trusted shop, but it does not scale to a shared platform:

- The webhook URL is visible to anyone who views source, so **anyone could POST to it**
  and send emails to a client's customers, impersonating the shop.
- No per-tenant routing — one hardcoded workflow.
- If the browser closes mid-request, the notification is silently lost.

On the platform, the database dispatches instead:

```
status change / staff message
        ↓  (Postgres trigger, SECURITY DEFINER)
insert row in public.order_messages         ← permanent log, append-only
        ↓  (trigger → pg_net)
POST  platform_settings.n8n_notify_url      ← URL never leaves the server
        ↓
n8n workflow  → routes on company_code → Gmail / Twilio
        ↓
pg_cron reconciles the real HTTP response → delivery = sent | failed
```

**The n8n URL is unreachable from any browser** — `platform_settings` has RLS on with
no policies and no grants, so both `anon` and signed-in users get
`permission denied`. Only the `SECURITY DEFINER` trigger reads it.

## What n8n receives

`POST` with `Content-Type: application/json`:

```jsonc
{
  "event": "status_update",        // or "custom"
  "message_id": 42,
  "company_code": "TEST-CO",       // ← route per tenant on this
  "company_name": "Relentless Motorsports TX",
  "company_config": {},            // companies.config jsonb (per-tenant settings)
  "order": {
    "order_code": "ORD-1004",
    "customer_name": "Tina Vasquez",
    "customer_email": "tvasquez@outlook.com",
    "customer_phone": "(512) 555-0177",
    "vehicle": "2019 Ford Mustang GT",
    "service": "Custom Tune",
    "status": "In Progress"
  },
  "channel": "email",
  "body": "Free-text message (custom messages only, null for status updates)",
  "from_status": "New",            // status_update only
  "to_status": "In Progress",      // status_update only
  "sent_by": "Test Owner",         // null when triggered by the public form
  "sent_at": "2026-07-16T00:03:17Z"
}
```

Two events:

| `event` | Fired when | Use |
|---|---|---|
| `status_update` | An order's status changes **and** `orders.notify_customer` is true **and** the customer has an email | "Your order is now In Progress" |
| `custom` | Staff types a message in Manage → **✉ Send** | Free-text update |

## Building the n8n workflow (this is the remaining step)

1. Add a **Webhook** node — method `POST`, path `platform-notify`.
2. **Activate the workflow** (an inactive workflow returns 404 on the production URL —
   that is exactly what the platform is getting today).
3. Route on `{{ $json.company_code }}` (Switch node) if different shops need different
   senders/branding; otherwise handle them all in one branch.
4. Branch on `{{ $json.event }}`:
   - `status_update` → compose from `order.status` / `to_status`
   - `custom` → send `body` verbatim
5. Send via Gmail to `{{ $json.order.customer_email }}` (or Twilio to `order.customer_phone`
   when `channel` is `sms`).
6. **Return 2xx.** Any non-2xx is recorded as `failed` with the status code and n8n's
   response body, and shown in the message history.

### Changing the endpoint

```sql
update public.platform_settings
set value = 'https://n8n.braxtonjeht.com/webhook/whatever'
where key = 'n8n_notify_url';
```
Takes effect on the next message — no deploy.

To disable notifications entirely, set it to `''` — messages get logged as `skipped`.

## Delivery states

| State | Meaning |
|---|---|
| `pending` | Row created, dispatch not attempted yet |
| `dispatched` | Handed to pg_net, awaiting n8n's response (shows as "sending…") |
| `sent` | **n8n returned 2xx** — the only state that means delivered |
| `failed` | n8n returned non-2xx, timed out, or errored (reason in `delivery_note`) |
| `skipped` | No customer email on file, or no endpoint configured |

`dispatched` deliberately does **not** display as "sent": pg_net is asynchronous, so
the HTTP result arrives later. `public.reconcile_message_delivery()` runs every minute
via pg_cron, reads `net._http_response`, and records the real outcome. Anything stuck
in `dispatched` for 5 minutes is marked failed.

## Verifying

```sql
-- what actually happened to each message
select id, kind, delivery, delivery_note, sent_by_name, created_at
from public.order_messages order by created_at desc limit 20;

-- raw HTTP responses from n8n
select id, status_code, left(content,200), created
from net._http_response order by id desc limit 10;
```

## Notes / limits

- Messages are **append-only from the client** — staff can insert a `custom` message for
  their own company's order, but cannot update or delete the log. `sent_by` is stamped
  from `auth.uid()` server-side, so a sender cannot be forged.
- The public intake form can create orders but never sends messages.
- Retries are not implemented — a `failed` message stays failed. Re-send by sending a
  new message. (Worth adding if n8n proves flaky.)
- `company_config` is passed through untouched so per-shop settings (from-name, reply-to,
  SMS opt-in) can live in `companies.config` without a schema change.
