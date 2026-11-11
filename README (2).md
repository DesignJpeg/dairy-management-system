# Dairy Desk

A mobile-first, single-file web app for small dairy businesses to track daily milk deliveries, generate monthly bills, record payments and see simple analytics. Built with plain HTML/CSS/JS and [Supabase](https://supabase.com) as the backend. No build step.

## Features

- **Today screen**: one-tap "All delivered as usual", per-customer +/- 250 ml buttons, "No milk" shortcut.
- **Quick entry**: type or speak (e.g. `Ekta 1 litre, Ravi half, Sunil no milk`). Names are matched on the device and never sent anywhere. Understands English, Hindi/Hinglish and Bengali words.
- **Register**: monthly per-customer register, morning/evening slots, cow/buffalo milk, per-customer price and default litres.
- **Offline-friendly staging**: edits are kept on the phone (localStorage) and written to the database only when you tap **Save all**.
- **Billing**: adjustments (discount / extra charge), lock a bill with a sequential invoice number (`INV-YYYY-001`), unlock (voids the number), month-end **Bill run** to lock and send all bills one by one.
- **Sharing**: PDF bill with UPI payment QR (jsPDF), Web Share, WhatsApp text and reminders.
- **Payments**: cash / UPI / bank transfer with running balance.
- **Analytics**: daily and monthly charts, top customers, highest unpaid, slow payers, customers losing milk, collection rate, estimated profit.
- **AI advice (optional)**: summaries and recommendations via Groq. Customer names are replaced by `Customer 1, 2, 3` before anything is sent.
- **Bilingual UI**: English and Bengali (bills and WhatsApp text stay English).
- **Backups**: weekly, monthly and full JSON export.

## Tech stack

- Vanilla JS, single `index.html`
- [Supabase](https://supabase.com) (Postgres, Auth, optional Edge Functions) via `@supabase/supabase-js`
- [jsPDF](https://github.com/parallax/jsPDF) + jspdf-autotable
- [qrcode-generator](https://github.com/kazuhikoarase/qrcode-generator)
- All libraries load from the jsDelivr CDN.

## Setup

### 1. Create a Supabase project

Create a project at supabase.com, then copy the **Project URL** and **anon public key** from *Project Settings > API*.

### 2. Configure the app

Open `index.html` and edit near the top of the main `<script>`:

```js
const SUPABASE_URL='YOUR_SUPABASE_URL', SUPABASE_KEY='YOUR_SUPABASE_ANON_KEY';
```

The anon key is designed to be public, but **only if Row Level Security is enabled** (see below). Never put the `service_role` key in this file.

### 3. Create the database

The app expects the following objects. Create them in the Supabase SQL editor.

**Tables** (column names as used by the code):

| Table | Columns |
|---|---|
| `settings` | `id` (single row, `id = 1`), `business_name`, `phone`, `address`, `upi_id`, `price_per_litre`, `price_buffalo`, `default_ml`, `footer`, `cost_cow`, `cost_buffalo`, `groq_key`, `groq_model` |
| `customers` | `id` (uuid), `name`, `phone`, `slots` (`morning`/`evening`/`both`), `milk_type` (`cow`/`buffalo`/`both`), `default_ml_cow`, `default_ml_buffalo`, `price_cow`, `price_buffalo`, `address`, `notes`, `active` (default `true`) |
| `milk_entries` | `id`, `customer_id`, `entry_date`, `slot`, `milk_type`, `ml`, `rate`, `updated_at`; **unique** on `(customer_id, entry_date, slot, milk_type)` |
| `payments` | `id`, `customer_id`, `paid_on`, `amount`, `method`, `note` |
| `adjustments` | `id`, `customer_id`, `adj_month` (first day of month), `kind` (`discount`/`extra`), `amount`, `note`, `created_at` |
| `invoices` | `id`, `invoice_no` (**unique**), `customer_id`, `bill_month` (first day of month), `status` (`locked`/`void`, default `locked`), `balance_paise`, `snapshot` (jsonb), `issued_on` (date, default today), `voided_at` |

**RPC functions** the app calls (you need to implement these in SQL):

- `prev_balance(cid uuid, before date)` returns the customer's balance (in paise) before a date.
- `an_cust_month(from_m date, to_m date)` returns JSON `{ rows: [{cid, m, ml, ch, ex, dc, paid}], opening: {cid: paise} }`.
- `an_month(from_d date, to_d date)` returns rows `{m, milk_type, slot, ml, p}`.
- `an_daily(from_d date, to_d date)` returns rows `{d, slot, milk_type, ml}`.
- `an_methods(from_d date, to_d date)` returns rows `{m, method, p}`.

> A complete `schema.sql` is not included yet. Contributions welcome.

To make locked bills immutable, add a trigger that rejects inserts/updates/deletes on `milk_entries` and `adjustments` for months with a locked invoice (the app already handles this error gracefully).

### 4. Security (important)

1. Enable **Row Level Security** on every table.
2. Add policies allowing access only to authenticated users, e.g. `using (auth.role() = 'authenticated')`.
3. In *Authentication > Users*, create your own user manually and **disable public sign-ups**, so only you can log in.

Without RLS, anyone who finds your anon key can read and write your data.

### 5. Optional: AI advice

Choose one:

- **Direct Groq key**: paste your key in *Settings*. It is stored in the `settings` table (not in the file), so anyone who can log in can read it. Browsers may block direct calls to Groq.
- **Supabase Edge Function** named `groq-advice` that accepts `{mode, lang, question, data}` and returns `{text}`, keeping the key as a server-side secret. Recommended.

Leave both empty to hide nothing, the feature simply reports it is unavailable.

### 6. Run / deploy

It is a static file. Open it locally or host it anywhere (GitHub Pages, Netlify, Cloudflare Pages, Vercel).

```bash
python3 -m http.server 8000
# open http://localhost:8000
```

## Privacy

- Voice typing uses the browser's speech service.
- Quick-entry name matching happens on the device.
- AI requests contain only aggregated numbers and anonymised `Customer N` labels, never names, phones or addresses.

## Contributing

Issues and pull requests are welcome.

## License

Add a license of your choice (e.g. MIT) before publishing.
